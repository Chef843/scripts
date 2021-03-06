## Table of Contents
* [Setup](#setup)
* [Signatures](#signatures)
* [Scripts](#scripts)
* [Logs](#logs)
* [Sniff](#sniff)
* [Correlate](#correlate)

## Setup
```bash
sudo apt install cmake make gcc g++ flex bison libpcap-dev libssl-dev python-dev swig zlib1g-dev
sudo apt install scapy
```

## Signatures
```bash
vim dns-intel.sig
```
```bash
# add to file
signature dns-intel {
	ip-proto == udp
	dst-port == 53
	payload /.*ooo|.*gdn|.*bar|.*work|.*life/
	event "[Suspicious DNS query] "
}
```

## Scripts
```bash
vim dns-alert.bro
```
```bash
event signature_match (state: signature_state, msg: string, data: string) {
	if (state$sig_id == "dns-intel") {
		print fmt ("[Suspicious DNS query] %s", state$conn$dns$query)
	}
}
```

## Logs
```bash
bro -r dns-traffic.pcap -s dns-intel.sig dns-alert.bro
```
```bash
ls *.log
cat dns.log
cat dns.log | head -n7 | tail -n1
cat dns.log | bro-cut ts id.orig_h id.orig_p query
```

## Sniff
```bash
echo '@load-sigs $PREFIX/share/bro/site/dns-intel.sig' | sudo tee -a $PREFIX/share/bro/site/local.bro
```
```bash
sudo broctl
[BroControl] > install
[BroControl] > start
```
```bash
sudo tcpreplay -i lo dns-traffic.pcap
[BroControl] > stop
[BroControl] > exit
```
```bash
zcat $PREFIX/logs/$DATE/signature$DATE.log.gz | bro-cut -u event_msg
```

## Correlate
```bash
vim detect-watering-hole-attack.bro
```
```zeek
# load Zeek's SMB framework of analyzers
@load policy/protocols/smb

# create some global variables
global hashes_all: set[string];
global hashes_downloaded_via_HTTP: set[string];
global hashes_uploaded_via_SMB: set[string];

# when you see a file
event file_new (f: fa_file) {

	# if the file was seen in HTTP or SMB traffic, hash it
	if (f$source == "HTTP" || f$source == "SMB") {
		Files::add_analyzer(f, Files::ANALYZER_MD5);
	}
}

# when you're asked to hash a file
event file_hash (f: fa_file, kind: string, hash: string) {

	# keep track of the hashed file
	add hashes_all[hash];
	
	# if the hashed file was seen in HTTP, track it as such
	if (f$source == "HTTP" && f$http$uri != "/") {
		add hashes_downloaded_via_HTTP[hash];
		print fmt ("[INFO] Connection: %s, Downloaded: %s, MD5: %s", f$http$uid, f$http$uri, hash);
	}

	# if the hashed file was seen in SMB, track it as such
	if (f$source == "SMB") {
		add hashes_uploaded_via_SMB[hash];
		
		# extract the connection uid
		for (con in f$conns) { local cid = f$conns[con]$uid; }
		
		# extract the connection data
		for (con in f$conns) { local cdata = f$conns[con]; }
		print fmt ("[INFO] Connection: %s, Uploaded: %s, MD5: %s", cid, f$info$filename, hash);

		# if the hashed file was seen in BOTH HTTP & SMB, alert me
		if (hash in hashes_downloaded_via_HTTP && hash in hashes_uploaded_via_SMB) {
			print fmt ("[ALERT] Possible watering hole attack in progress.");
			print fmt ("   -->  %s, MD5: %s", f$info$filename, hash);

			# and send the following info to 'notice.log'
			NOTICE([
				$note = Weird::Activity, 
				$msg = "Possible watering hole attack in progress.",
				$conn = cdata
			]);
		}
	}
}
```
```bash
bro -r traffic.pcap detect-watering-hole-attack.bro
cat notice.log | bro-cut -u ts uid msg
```
