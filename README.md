<h1>Detecting Nmap Scans with Wireshark</h1>

<p>
This project demonstrates how different <strong>Nmap scan techniques</strong> can be identified
by analyzing packet behavior in <strong>Wireshark</strong>.
The goal is to understand how port states (open, closed, filtered)
manifest at the packet level and how a SOC analyst can recognize active reconnaissance.
</p>

<hr>

<h2>Lab Environment</h2>
<ul>
  <li><strong>Kali Linux</strong> (10.0.0.2) — target, Wireshark packet capture</li>
  <li><strong>Metasploitable 2</strong> (10.6.6.11) — attacker </li>
  <li><strong>pfSense</strong> — Inter-network routing and firewall rules</li>
</ul>

<p>
Metasploitable 2 initiates various Nmap scans against Kali.
Wireshark is used on Kali to capture and analyze the resulting traffic.
</p>

<hr>

<h2>Wireshark TCP Flag Filters Reference</h2>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th>Filter</th>
    <th>Description</th>
  </tr>
  <tr><td>tcp.flags == 1</td><td>Only FIN flag set</td></tr>
  <tr><td>tcp.flags == 2</td><td>Only SYN flag set</td></tr>
  <tr><td>tcp.flags == 4</td><td>Only RST flag set</td></tr>
  <tr><td>tcp.flags == 16</td><td>Only ACK flag set</td></tr>
  <tr><td>tcp.flags == 18</td><td>SYN + ACK flags set</td></tr>
  <tr><td>tcp.flags == 20</td><td>RST + ACK flags set</td></tr>
  <tr><td>tcp.flags.syn == 1</td><td>SYN flag set (others ignored)</td></tr>
  <tr><td>tcp.flags.reset == 1</td><td>RST flag set (others ignored)</td></tr>
  <tr><td>tcp.flags.ack == 1</td><td>ACK flag set (others ignored)</td></tr>
  <tr><td>tcp.flags.fin == 1</td><td>FIN flag set (others ignored)</td></tr>
  <tr><td>(tcp.flags.syn == 1) and (tcp.flags.ack == 1)</td><td>SYN and ACK flags set</td></tr>
</table>

<hr>

<h2>ICMP Detection Filters</h2>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th>Filter</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>icmp.type == 3 and icmp.code == 3</td>
    <td>ICMP Port Unreachable (UDP closed port)</td>
  </tr>
</table>

<hr>

<h2>TCP SYN Scan (<code>nmap -sS</code>)</h2>

<h3>Overview</h3>
<p>
The TCP SYN scan is Nmap’s most popular scan type.
It is fast, stealthy, and does not complete the TCP three-way handshake.
</p>

<h3>Packet Flow</h3>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th colspan="2">Open Port</th>
    <th colspan="2">Closed Port</th>
  </tr>
  <tr>
    <td>Attacker</td><td>Target</td>
    <td>Attacker</td><td>Target</td>
  </tr>
  <tr>
    <td>SYN →</td><td></td>
    <td>SYN →</td><td></td>
  </tr>
  <tr>
    <td></td><td>← SYN, ACK</td>
    <td></td><td>← RST, ACK</td>
  </tr>
  <tr>
    <td>RST →</td><td></td>
    <td></td><td></td>
  </tr>
</table>

<h3>Scenario</h3>

<pre>
Kali: open port 22 (SSH)
sudo systemctl start ssh

Metasploitable 2:
nmap -sS 10.0.0.2
</pre>

<h3>Wireshark Analysis</h3>
<ul>
  <li>RST packets: <code>tcp and tcp.flags == 4</code></li>
  <li>RST + ACK packets: <code>tcp and tcp.flags == 20</code></li>
</ul>

<p><strong>Open port (22 / SSH):</strong></p>
<img width="1980" height="220" alt="syn open" src="https://github.com/user-attachments/assets/f52ae3de-1f51-4f5c-baaa-0e6b83ecf731" />


<p><strong>Closed port (23):</strong></p>
<img width="2002" height="200" alt="syn closed" src="https://github.com/user-attachments/assets/c6467e23-ea12-47d2-809f-becb8b1a8449" />

<hr>

<h2>TCP Connect Scan (<code>nmap -sT</code>)</h2>

<h3>Overview</h3>
<p>
Used when SYN scanning is not possible.
This scan completes the full TCP three-way handshake.
</p>

<h3>Packet Flow</h3>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th colspan="2">Open Port</th>
    <th colspan="2">Closed Port</th>
  </tr>
  <tr>
    <td>Attacker</td><td>Target</td>
    <td>Attacker</td><td>Target</td>
  </tr>
  <tr>
    <td>SYN →</td><td></td>
    <td>SYN →</td><td></td>
  </tr>
  <tr>
    <td></td><td>← SYN, ACK</td>
    <td></td><td>← RST, ACK</td>
  </tr>
  <tr>
    <td>ACK →</td><td></td>
    <td></td><td></td>
  </tr>
  <tr>
    <td>RST, ACK →</td><td></td>
    <td></td><td></td>
  </tr>
</table>

<h3>Scenario</h3>

<pre>
Kali: open port 22 (SSH)
sudo systemctl start ssh

Metasploitable 2:
nmap -sT 10.0.0.2
</pre>

<h3>Wireshark Analysis</h3>
<p>
Filter for RST + ACK packets:
</p>
<pre>
tcp and tcp.flags == 20
</pre>

<ul>
  <li>If RST+ACK originates from the attacker → port is open</li>
  <li>If RST+ACK originates from the target → port is closed</li>
</ul>

<p><strong>Open port (22 / SSH):</strong></p>
<img width="2396" height="226" alt="tcp open" src="https://github.com/user-attachments/assets/b7fe3209-7e4b-4486-b418-9bd476fd81b9" />


<p><strong>Closed port (23):</strong></p>
<img width="2390" height="182" alt="tcp closed" src="https://github.com/user-attachments/assets/e74b5f8d-f5e6-4e7a-a264-362135e3a3d3" />

<hr>

<h2>UDP Scan (<code>nmap -sU</code>)</h2>

<h3>Overview</h3>
<p>
Checks UDP ports (including 53 – DNS, 161/162 – SNMP, 67/68 – DHCP). Slower and more difficult than the TCP scan. Most open ports do not respond, and nmap resends the request in case the first one was lost. Moreover, many hosts rate-limit the ICMP Port Unreachable messages by default. Nmap detects rate limiting and slows down accordingly, which can make the UDP scan very slow.
</p>

<h3>Packet Flow</h3>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th colspan="2">Open Port</th>
    <th colspan="2">Closed Port</th>
  </tr>
  <tr>
    <td>Attacker</td><td>Target</td>
    <td>Attacker</td><td>Target</td>
  </tr>
  <tr>
    <td>UDP →</td><td></td>
    <td>UDP →</td><td></td>
  </tr>
  <tr>
    <td></td><td>← UDP response (optional)</td>
    <td></td><td>← ICMP Port Unreachable</td>
  </tr>
</table>

<h3>Scenario</h3>

<pre>
pfSense:
Allow UDP traffic from 10.6.6.11 to 10.0.0.2 on port 9999

Kali: set up a listener on port 9999
sudo socat -v UDP-LISTEN:9999,fork STDOUT

Metasploitable 2:
nmap -sU -p 9998-9999 10.0.0.2
</pre>

<h3>Wireshark Analysis</h3>

<p>Filter for ICMP Port Unreachable:</p>
<pre>
icmp.type == 3 and icmp.code == 3
</pre>

<p>
Other ICMP unreachable codes (0,1,2,9,10,13) indicate filtered ports.
No response after retransmission results in an open|filtered classification.
</p>

<p><strong>Open port (9999):</strong></p>
<img width="2136" height="200" alt="udp open" src="https://github.com/user-attachments/assets/081571aa-8caa-4a9c-920a-d12b55d7da07" />

<p><strong>Closed port (9998):</strong></p>
<img width="2140" height="192" alt="udp closed" src="https://github.com/user-attachments/assets/af35a6cb-43d3-4613-8c71-2acc73bff9fd" />

<hr>
<h2>FIN, NULL, XMAS Scans (<code>nmap -sF, -sN, -sX</code>)</h2>

<h3>Overview</h3>
<p>
Any packet not containing SYN, ACK or RST flag will result in no response from the open port and RST packet from the closed port. These scan types utilize this by sending no bits at all (TCP header is 0) for NULL scan, FIN flag for FIN scan, and FIN, PSH, and URG flags for Xmas scan. 
Modern systems often drop unsolicited packets not containing SYN, ACK, RST flags, so these scans might result is showing all ports open since no response would be received.
</p>

<h3>Packet Flow</h3>

<table border="1" cellpadding="6" cellspacing="0">
  <tr>
    <th colspan="2">Open Port</th>
    <th colspan="2">Closed Port</th>
  </tr>
  <tr>
    <td>Attacker</td><td>Target</td>
    <td>Attacker</td><td>Target</td>
  </tr>
  <tr>
    <td>SYN →</td><td></td>
    <td>SYN →</td><td></td>
  </tr>
  <tr>
    <td></td><td></td>
    <td></td><td>← RST, ACK</td>
  </tr>
 </table>

<h3>Scenario</h3>

<pre>
Kali: open port 22 (SSH)
sudo systemctl start ssh

Metasploitable 2:
nmap -sF 10.0.0.2
nmap -sN 10.0.0.2
nmap -sX 10.0.0.2
</pre>

<h3>Wireshark Analysis</h3>
<p>
When the scan was conducted from Metasploitable2 to Kali Linux, the Wireshark at Kali logged no packets at all. Since Meta received no response, it marked all scanned ports as open|filtered.
</p>

<hr>

<h2>Skills Demonstrated</h2>
<ul>
  <li>Network traffic analysis with Wireshark</li>
  <li>Detection of reconnaissance activity</li>
  <li>Understanding TCP/UDP port states</li>
  <li>Nmap scan fingerprinting</li>
  <li>SOC-level packet interpretation</li>
</ul>
