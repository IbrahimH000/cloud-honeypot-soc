1. Overview
	- This playbook documents the detection, triage, and remediation phases for SSH brute force attacks identified by GuardDuty against EC2 instances. The finding type shows that an EC2 instance is being targeted by repeated SSH login attempts from external IP addresses using automated credential stuffing or dictionary attacks
	- Within 24 hours of deploying a cowrie SSH honeypot, the environment recorded 4039 connection attempts from 7 unique IPs, all targeting the root account. 
2. Finding Details
	- 
3. Detection Pipeline
4. ````
   Attacker → EC2 Honeypot (port 2222)
               ↓
         Cowrie logs attack to cowrie.json
               ↓
         VPC Flow Logs capture traffic
               ↓
         GuardDuty analyzes + matches threat intel
               ↓
         Finding generated → EventBridge rule fires
               ↓
         SNS topic → Email alert to analyst
````
4. Triage steps
	- Step 1: Confirm finding
		- Log in to GuardDuty --> Findings
		- Review the finding details
		- Check if the source IP is a known scanner using threat (VirusTotal, Abuse IPDB)
		- Determine if this is the honeypot instance (expected) or production instance 
	- Step 2: Assess severity
	- Step 3: Gather Evidence
		- Check Cowrie logs for attacker behaviour 
			- `cd ~/cowrie 
			- `cat var/log/cowrie/cowrie.json | grep "src_ip":"ATTACKER_IP"`
		- Get total attempts from that
			- `IP cat var/log/cowrie/cowrie.json |grep "ATTACKER_IP" | wc -l` 
		- Check what commands attacker ran if they got in 
			- `cat var/log/cowrie/cowrie.json | grep "command"`
	- Step 4: Look up attacker IP
		- Search up the IP on AbuseIPDB and VirusTotal
		- Note the country of origin and if it appears on known threat lists
		- Document findings, guardDuty will flag IPs it matches against intel feeds
5. Containment
	- If honeypot instance (expected attack traffic):
		- No containment needed - intended behaviour
		- Update security group to block specific IPs if you want to stop it from being logged
	- If production instance (unexpected)
		- Isolate the instance immediately
		- Revoke any active sessions
		- Rotate SSH keys and credentials on the instance
		- Snapshot the instance image before any remediation for forensic preservation 
6. Investigation: Run these to get a full image of the attack
	- Total connections
		- `cat var/log/cowrie/cowrie.json | grep "session.connect" | wc -l`
	- All unique attacker IPs
		- `cat var/log/cowrie/cowrie.json | grep -o '"src_ip":"[^"]*"' | sort -u`
	- Targeted usernames
		- `cat var/log/cowrie/cowrie.json | grep -o '"username":"[^"]*"' | sort | uniq -c | sort -rn | head -10`
	- Passwords attempted
		- `cat var/log/cowrie/cowrie.json | grep -o '"password":"[^"]*"' | sort | uniq -c | sort -rn | head -10`
	- Any commands run by attackers who got past authentication. 
		- `cat var/log/cowrie/cowrie.json | grep "cowrie.command.input"`
7. Remediation & Hardening Recommendations
	- Based on the observed attack behaviours
		- Never expose SSH port 22 to the public as it can be found within minutes and used. 
		- Disable root login 100% of observed attacks
		- Use key based authentication, every attacker used passwords to attempt to log in. Key-based auth renders these attacks useless 
		- Implement fail2ban to ban certain IPs after N failed attempts 
8. Takeaways
	- Internet-exposed SSH is targeted immediately and constantly
	- Default usernames are the first vector attackers try
	- Attacks are automated, distributed, and use large wordlists
	- Attackers use scripted tools, not manual attempts