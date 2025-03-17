```ini
					---- dnsdist configuration file, an example can be found in /usr/share/doc/dnsdist/examples/
---- disable security status polling via DNS
setSecurityPollSuffix("")

---- Chỉ định địa chỉ IP và port của máy chủ web
webserver("10.10.40.30:8083")
---- Chỉ định địa chỉ IP cho phép truy cập web
setWebserverConfig({acl="192.168.2.10/32"})

---- Chỉ định địa chỉ IP, port và setKey của control socket
controlSocket('127.0.0.1:5199')
setKey("2ux3QDmpdDAzYjspjkhjkuyF8jXFU5qhd/BqXV8ag=")

---- Cấu hình ghi log
---- addAction(AllRule(), LogAction("/var/log/dnsdist.log", false, true, false,true))

---- Cấu hình lắng nghe truy vấn DNS qua giao thức UDP/53 and TCP/53
setLocal('0.0.0.0:53')

---- Cấu hình lắng nghe truy vấn DNS qua giao thức DoT (DNS over TLS) queries on TCP/853
addTLSLocal('0.0.0.0:853', {"/etc/dnsdist/ssl/vinahost.vn.cert.combined"}, {"/etc/dnsdist/ssl/vinahost.vn.key"})

---- Cấu hình lắng nghe truy vấn DNS qua giao thức DNS over HTTPS (DoH) queries on TCP/443
addDOHLocal("0.0.0.0:443", {"/etc/dnsdist/ssl/vinahost.vn.cert.combined"}, {"/etc/dnsdist/ssl/vinahost.vn.key"}, "/dns-query")

---- Định nghĩa các máy chủ back-end

newServer({address="10.10.10.1",name="nsmaster", weight=1, tcpFastOpen=true, healthCheckMode='lazy', checkInterval=1, lazyHealthCheckFailedInterval=30, rise=2, maxCheckFailures=3, lazyHealthCheckThreshold=30, lazyHealthCheckSampleSize=100,  lazyHealthCheckMinSampleCount=10, lazyHealthCheckMode='TimeoutOnly'})
newServer({address="10.10.20.1",name="ns-slave", weight=2, tcpFastOpen=true, healthCheckMode='lazy', checkInterval=1, lazyHealthCheckFailedInterval=30, rise=2, maxCheckFailures=3, lazyHealthCheckThreshold=30, lazyHealthCheckSampleSize=100,  lazyHealthCheckMinSampleCount=10, lazyHealthCheckMode='TimeoutOnly'})

---- Định nghĩa chính sách xử lý truy vấn
setACL({'0.0.0.0/0', '::/0'})
---- Cấu hình policy load balancing đến Backend
setServerPolicy(wrandom) -- weight random Khi có 3 yêu cầu đến thì ns-slave sẽ trả lời 2 query, nsmaster sẽ trả lời 1
---- Chặn và phản hồi các query có type là any
addAction(QTypeRule(DNSQType.ANY), TCAction())

---- Cấu hình Dynamic Blocking Rules (DBR)
local dbr = dynBlockRulesGroup()
---- Rate limit 30 query trong 10 giây, sẽ chặn trong 60 giây nếu đạt giới hạn
dbr:setQueryRate(30, 10, "Exceeded query rate", 60)
---- Rate limit 20 query với NXDOMAIN ( non-existent domain (tên miền không tồn tại)) trong 10 giây, sẽ chặn trong 60 giây nếu đạt giới hạn
dbr:setRCodeRate(DNSRCode.NXDOMAIN, 20, 10, "Exceeded NXD rate", 60)
---- Rate limit 20 query với SERVFAIL trong 10 giây, sẽ chặn trong 60 giây nếu đạt giới hạn
dbr:setRCodeRate(DNSRCode.SERVFAIL, 20, 10, "Exceeded ServFail rate", 60)
---- Rate limit 5 query với ANY (tất cả type) trong 10 giây, sẽ chặn trong 60 giây nếu đạt giới hạn
dbr:setQTypeRate(DNSQType.ANY, 5, 10, "Exceeded ANY rate", 60)
---- Rate limit BW 10000 byte query trong 10 giây, sẽ chặn trong 60 giây nếu đạt giới hạn
dbr:setResponseByteRate(10000, 10, "Exceeded resp BW rate", 60)

---- Định nghĩa hàm "maintenance"
function maintenance()
    dbr:apply()
end


--- Định nghĩa để xóa bộ nhớ cache
function flushCache(dq)
    errlog("Got notify for " .. dq.qname:toString())
    a = getPool(""):getCache():toString()
    getPool(""):getCache():expungeByName(dq.qname, DNSQType.ANY, true)
    z = getPool(""):getCache():toString()
    errlog("Went from " .. a .. " to " .. z)
    return DNSAction.None
end

---- Sao chép Notify tới nhiều back-end
addAction(AndRule({OpcodeRule(DNSOpcode.Notify)}), TeeAction("10.10.10.1"))
addAction(AndRule({OpcodeRule(DNSOpcode.Notify)}), TeeAction("10.10.20.1"))


-- Xóa bộ nhớ cache nếu back-end trả về NOERROR
addResponseAction(AndRule({OpcodeRule(DNSOpcode.Notify), RCodeRule(DNSRCode.NOERROR)}), LuaResponseAction(flushCache))

---- Cấu hình bộ nhớ cache
pc = newPacketCache(1000000, {maxTTL=300, minTTL=0, staleTTL=60, dontAge=false})
getPool(""):setCache(pc)
```