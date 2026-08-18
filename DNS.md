how DNS work
1. when user enters google.com ->
	1. browser cache if not found
	2. OS cache if not found then goes to DNS resolver
	3. DNS resolver cache if not found
	4. Root nameserver -> it find .com nameserver
	5. then it goes to TLD (top level domain) name server -> it find authoritative nameserver for google.com
	6. then it goes to authoritative nameserver => which returns IP address of google.com
	7. then goes to DNS resolver
	8. then OS
	9. then Browser