##DEF CON 23 OpenCTF
###ffbank

I recently played in the [OpenCTF](http://www.openctf.com) at DEF CON 23 alongside team_reddit, which formed on [/r/defcon](https://reddit.com/r/defcon) just a few days before the competition. For many of us (myself included) it was one of our first CTFs, and we managed to come in 12th place out of about 60 teams. I had a great time playing in it. Many thanks to dook, the rest of the team, and especially to v& for putting the whole thing together.

We came excruciatingly close to solving ffbank (100pts, raised to 250 mid-competition), and we probably would have gotten the flag if we managed to start on it just a bit earlier. Still, it was a great challenge and taught me a lot about VLANs, VoIP pivoting, and scripting. Thanks to Kajer for designing this problem and helping us try to figure it out!

Starting the challenge, the server tells us "your pen testing friend just sent you this from a recent job." Looking at the challenge file, we have a pcap with quite a bit of interesting traffic.

![Excerpt from the ffbank pcap](https://raw.githubusercontent.com/jkotheim/ctf-writeups/master/dc23-openctf/images/ffbank1.png)

Looking through the pcap, we initially see a lot of TCP and STP packets, as well as a few other protocols (NTP and CDP). Let's have a look at the TCP first. Following the stream in Wireshark, we can easily see some plaintext traffic with what looks like some kind of bank system.


    25-Jul-2015   14:07:52
	/-------------------------------------------------/
	/--- Welcome to FFBANK teller services         ---/
	/---                                           ---/
	/---                         Code Rev. 13.3(7) ---/
	/---                                           ---/
	/--- Current Flag Value: 164690216             ---/
	/-------------------------------------------------/
	Please login below:


Wireshark tells us that the service of interest is running at `10.11.50.8:31337`, which is on our network, albeit a different /24. When we try to `nc` to the server, however, we are unsuccessful, and pings get filtered out. Let's investigate further.

After thinking about it for a while (and possibly getting a hint or two from the organizers) we realized that the challenge server is on a different VLAN from us. Investigating the pcap further shows this: there are a ton of STP packets all showing that they are on a VLAN with ID 1150. (See below for an aside on another method to access this VLAN.)

We created a new network interface tagged to this VLAN and statically assigned it an IP, as follows:


	root@kali# modprobe 8021q
	root@kali# ip link add link eth0 name eth0.1150 type vlan id 1150
	root@kali# ip addr add 10.11.50.23/24 brd 10.11.50.255 dev eth0.1150
	root@kali# ip link set dev eth0.1150 up


When we `netcat` to the challenge server, we get a response! Now we can log in with the credentials revealed in the pcap and try to get the flag. When we logged in, we got the following options:


	1) List accounts and balance
	2) Internal transfer
	3) External transfer
	4) Customer Deposit
	5) Customer Withdrawl
	6) Open new account
	7) Close an account
	9) Obtain Flag
	0) Manager Deposit


You can see an example of all the different options in the pcap excerpt above, but what it looks like we have to do is get enough money into an account we control in order to purchase the flag for the value listed in the banner. Each account has a name, number, and automatically-generated, static key (obtained only at account creation.)

	Account Number: 00001001  Account Owner: John Smith           Balance: 90
	Account Number: 00000420  Account Owner: Mary Jane            Balance: 420
	Account Number: 43548459  Account Owner: Kirk Johnson         Balance: 2140972288
	Account Number: 00000023  Account Owner: Ayy LMAO             Balance: 10

Looking through the pcap and poking around on the server ourselves, we found 3 possible ways of raising the funds.

1. Write a script to keep making customer deposits, which are basically unauthenticated but capped at $100 per deposit. This would take an incredibly long time (given the flag costs about $170 million) so it's not ideal.
2. Write a script to keep making manager deposits, which are authenticated with a weird password (see below) but let us deposit $9999 at a time, giving us the flag about a hundred times faster than method 1.
3. Figure out how to crack the account keys and use funds from an existing account.

Since we were already running out of time when we got this far, method 1 was a non-starter, and we were not making a lot of progress figuring out how the account keys were generated, so we opted for method 2.

When we tried to make a manager deposit, we were challenged with a password similar to the following (from the pcap):


	Please enter the manager password
	HINT 06562: sitar ____ sitcoms ____ ______


Every time we logged into the manager deposit, we got a similar challenge with a different hint number and words, with the blanks in various places. This was not a lot to go off of, so we wrote a quick script to grab a ton of them to analyze with the hope that we could find a way to automatically crack the password.

To crack the password generation algorithm, we have a few pieces of data to work with:

* The words we are given
* The location of the blanks
* The number of the hint

We thought about how all of this information fit together, and realized that in all the hints, the words were close to one another alphabetically. We also noticed that there were always 5 words and/or blanks in total, as above. The hint number is also always 5-digits.

It turns out that the hint is actually a series of indices relative to an arbitrary word in a wordlist. This 'base word' has an index of zero, and we can infer what the other words are based on the index in the hint. To tie this to the example above, `sitar` is the 'base word', since we are told it is index zero, meaning `sitcoms` is five positions past it in the wordlist. These five words together make the manager password for that session. 

After much frustrated searching, we discovered that the wordlist used to generate these passwords appeared to be the British English wordlist from `/usr/share/dict`. (Did I mention that *nobody* solved this challenge?) We grabbed the `wbritish` package from the Kali repos and got cracking. Using this wordlist and the indexing, we found that the password to the above challenge would be `sitar site sitcoms site sitars`.

Once we tested a couple successful authentications, we started on a script to automate this, so we could raise some money to buy the flag. (I unfortunately do not have a copy of the source, but if someone else on the team still does, I will update this writeup to include it.)

Basically, our script read the entire dictionary into a data structure that we could navigate by index. We then got a hint off the challenge server, parsed the hint number and blanks, and used those to find the 'base word' for that password prompt. It was then simple to craft a password based on that info and feed it into the server, which then let us make a $9999 deposit.

We had just fired up the script and were raking in the dough when OpenCTF ended! It was disappointing to be so close to the flag after all that work, but it was definitely rewarding to get within sight of the end and nearly become the only team to solve that challenge.

We learned some great CTF techniques from this challenge and I personally got a lot out of it, being one of my first CTFs ever. Perhaps the most important lesson here is time management and tradecraft. On a complex, multi-staged challenge such as this one, you need to have a game plan and keep moving forward at all times. 

As we saw, there were multiple techniques to get to the flag, which employed different skill sets. It was a good lesson in knowing your own abilities, working well with your teammates, and pursuing not neccessarily the simplest, but the most efficient strategy.

Overall 10/10, would almost capture the flag again.

####Aside on VLAN traversal:

While we were able to connect to the challenge server simply by tagging it with the appropriate VLAN ID, there was another way to access the same network that utilized the CDP packets mentioned in the pcap file that we learned about from a conversation we had with the challenge designers. These packets appeared to come from a Cisco IP Phone 7940.

While we did not do this, it would have been possible to access the challenge server VLAN by spoofing our own CDP packets to associate ourselves with the VoIP network, whose packets were not restricted by VLAN. The `voiphopper` tool in Kali allows you to do this, and had we gone that route, we could have theoretically accessed the challenge server without creating the VLAN interface discussed above.
