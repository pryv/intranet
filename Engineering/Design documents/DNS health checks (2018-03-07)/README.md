|              |                       |
| ------------ | --------------------- |
| Author       | Thiébaud Modoux (Pryv) |
| Version      | 1 (07.03.2018)         |
| Distribution | Internally            |

# Situation

We had a customer reporting two main flaws in our DNS, thanks to a vulnerabilities scanner ([Nessus](https://www.tenable.com/products/nessus/nessus-professional)): 

- Our DNS is not recursive but do not warn about that fact
- Our DNS answers to DNS responses

During the investigations for these two issues, we computed a list of other warnings coming from different vulnerabilities scanners.

# DNS warnings list

## https://dnsspy.io/scan/pryv.me
- No IPv6 reachable nameservers were found. Users on IPv6-only networks are unable to reach you.
- No DNSSEC records found. Consider enabling DNSSEC, as it provides a way to validate DNS responses for data integrity.
- All the nameservers are being operated from a single domain (pryv.net). If that domain gets compromised or goes offline, the DNS will be unavailable. Consider spreading the nameservers across multiple domains.
- Consider giving the NS records a longer TTL, as those don't change often (1d+).

## https://intodns.com/pryv.me
- WARNING: SOA MNAME (reg.pryv.io) is not listed as a primary nameserver at your parent nameserver!

## http://dnscheck.pingdom.com/?domain=pryv.me
- Name server dns1-pryv-me.pryv.net (155.133.129.177) does not answer queries over TCP. The name server failed to answer queries sent over TCP. This is probably due to the name server not correctly set up or due to misconfgured filtering in a firewall. It is a rather common misconception that DNS does not need TCP unless they provide zone transfers - perhaps the name server administrator is not aware that TCP usually is a requirement.
- Name server dns1-pryv-me.pryv.net (155.133.129.177) is recursive.
The name server answers recursive queries for 3rd parties (such as DNSCheck). By making a recursive query to a name server that provides recursion, an attacker can cause a name server to look up and cache information contained in zones under their control. Thus the victim name server is made to query the attackers malicious name servers, resulting in the victim caching and serving bogus data.
- Host name reg.pryv.io refers to a CNAME.
The host name is an alias (CNAME), which is not allowed. Host names must be published with an A or AAAA record.
- Error while checking SOA MNAME for pryv.me (reg.pryv.io).
The SOA MNAME was not a valid host name.

## http://www.dnsstuff.com/tools#dnsReport|type=domain&&value=pryv.me
- Parent zone provides NS records: Parent zone does not provide glue for nameservers, which will cause delays in resolving your domain name. The following nameserver addresses were not provided by the parent 'glue' and had to be looked up individually. This is perfectly acceptable behavior per the RFCs. This will usually occur if your DNS servers are not in the same TLD as your domain (for example, a DNS server of "ns1.example.org" for the domain "example.com"). In this case, you can speed up the connections slightly by having NS records that are in the same TLD as your domain.
- Open DNS servers: Some or all nameservers responded to recursive queries. This should be addressed as soon as possible. Open DNS servers (i.e. externally facing DNS servers that answer recursively) increase the chances of cache poisoning, can degrade performance of your DNS, and can cause your DNS servers to be used in an attack (read RFC5358 section 4 for recommended nameserver configuration). The nameservers that responded to recursive queries are:
dns2-pryv-me.pryv.net. | 34.201.215.124 [test]
dns1-pryv-me.pryv.net. | 155.133.129.177 [test]
- TCP allowed: Not all nameservers responded to queries via TCP. This means that queries that require TCP connections will get inconsistent answers, which can cause delays or intermittent failures. The nameservers that failed TCP queries are:
dns2-pryv-me.pryv.net. | 34.201.215.124
dns1-pryv-me.pryv.net. | 155.133.129.177
- SOA field check	One or more SOA fields are outside recommended ranges. Values that are out of specifications could cause delays in record updates or unnecessary network traffic.
- Acceptance of postmaster: Mailserver rejected mail to postmaster. Mailservers are required by RFC822 6.3, RFC1123 5.2.7, and RFC2821 4.5.1 to have a valid postmaster address that is accepting mail. 
- Acceptance of abuse: Mailserver rejected mail to abuse. Mailservers are required by RFC2142 Section 2 to have a valid abuse address that is accepting mail.

## https://mxtoolbox.com/domain/pryv.me/
- dmarc 	pryv.me 	DNS Record not found
- mx 	pryv.me 	No DMARC Record found
- dns 	pryv.me 	Primary Name Server Not Listed At Parent
- spf 	pryv.me 	No SPF records found as TXT. (Record type 99 (SPF) has been deprecated)
- dns 	pryv.me 	SOA Expire Value out of recommended range 

## https://www.dnsqueries.com/en/domain_check.php
- Nameservers accept recursive queries. This can cause an excessive load on your DNS server and they can be used as part of an attack, by forging their IP address.
- Some servers are not accepting TCP connections. TCP connections are occasionally used instead of UDP connections and can be blocked by firewalls. This can cause hard-to-diagnose problems.
- Not all your nameservers agree on the identification of the primary nameserver or it isn't listed in the parent zone nameserver.
- The SOA REFRESH value determines how often secondary nameservers check with the master nameserver for updates.Your SOA REFRESH value is 1800 seconds which is too low (about 3600-7200 seconds is good althought RFC1912 2.2 recommends a value between 1200 to 43200 seconds). This is a warning, as this value can cause excessive network usage for zone transfers, but, if you are not paying for those transfers, you can ignore this alert.
- The expire value is how long a secondary nameserver will wait before considering its DNS data stale if it can't reach the primary nameserver. Your SOA EXPIRE value is 604800 seconds which is very low (as suggested by RFC1912 a value between 1209600 to 2419200 seconds is good).
- I have searched for differing IPs for your MX records between what are declaring your NS and the authoritative NS for the MX records. The check failed.
- Not all of your mailservers accept mail to postmaster@pryv.me.. RFC822 6.3, RFC1123 5.2.7, and RFC2821 4.5.1 require all mailservers to accept mail to this kind of address. 
- Not all of your mailservers accept mail to postmaster@pryv.me.. RFC822 6.3, RFC1123 5.2.7, and RFC2821 4.5.1 require all mailservers to accept mail to this kind of address. 
- Not all of your mailservers accept mail to postmaster@[ip_address] (Literal format). RFC1123 5.2.17 require all mailservers to accept mail to this kind of address. This is a common problem and actually can be ignored.
- You don't have a SPF record for the domain pryv.me., meaning that you are not using the protection given from this kind of technology. This is only a warning, but please consider in implementing it!
- I have asked your DNS server for www.pryv.me. but i did not receive an IP address (maybe i received a CNAME...), however these are the records i received: www.pryv.me. = CNAME w.pryv.com.
- There is one or more CNAMEs record pointing to www.pryv.me.. This can cause extra bandwidth usage since the resolution of www.pryv.me. is done in multiple steps. However this is only a warning!