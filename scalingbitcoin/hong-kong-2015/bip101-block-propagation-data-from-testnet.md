---
title: Bip101 Block Propagation Data From Testnet
TranscriptBy: Bryan Bishop
---

bip101 block propagation data from testnet

jtoomim

video: <https://www.youtube.com/watch?v=ivgxcEOyWNs&t=2h25m20s>

I am a bitcoin miner. I am a C++ programmer and a scientist. I will be going pretty fast. I have a lot of stuff to cover. Bare with me.

My perspective on this is that scaling bitcoin is an engineer problem. My favorite proposal for how to scale bitcoin is [bip101](https://github.com/bitcoin/bips/blob/master/bip-0101.mediawiki). It's over a 20-year time span. This will give us time to implement fixes to get Bitcoin to a large level. A hard-fork to increase the block size limit is hard, and soft-forks make it easier to decrease, that is to increase it is, so we should have a hard-fork once or at least infrequent and then do progressive short-term limitations when we run into scaling limitations.

In theory, well, first let me say bip101 is 8 MB now, and hten doubling every 2 years for the next 20 years, eventually reaching 8192 MB (8 GB). This is bip101. That 8 GiB would in theory require a large amount of computing power. 16 core CPU with 3.2 GHz per core. About 5000 sigops/sec/core. We would need about 256 GB of RAM. And you would need 8 GB every 64 seconds over the network. The kicker is the block propagation. With 1 gigabit internet connection, it would take 1 minute to upload an 8 GB block. You could use IBLT or other techniques to transmit a smaller amount of data at the time that the block is found, basically the header and some diffs.

There's some problems. Particularly the case with block propagation code. The RAM usage is not too bad right now. CPU is okay. It's going to get a lot better when libsecp256k1 is released. The block propagation itself is in my opinion really bad.

The effects of bad block propagation rates have been mentioned by petertodd and others, the quick idea is that currently China has 65% of the hashrate. Not all of that hashrate is running on servers in China. Most of the Chinese mining pools have servers all around the world. Some of it is abroad and some of it is in China. The Great Firewall is a big performance problem. The purpose of the Great Firewall is to censor and control. That has some problems for Bitcoin. It's uncomfortable to have so much hashrate subject to that system. Propagation over the Great Firewall is asymmetrical.

We wanted to figure out how big of a problem this block propagation thing is. We figured that Gavin had already used some tests with regtest, which is like testnet except predictable. I wanted to try to make things unpredictable, so we chos eto use testnet. We got 21 nodes under our control. 10 under my control, 11 from various users on reddit. Most of these were mid-range VPS, we had a few vastly-underpowered machines, and we also had some boxes running Intel atom CPUs. Our data collection method was different from others for [measuring block propagation, lightsword and gmaxwell used stratum to see when a block is being offered by a mining pool](http://diyhpl.us/wiki/transcripts/gmaxwell-2015-11-09-mining-and-block-size-etc/). To monitor the debug logs, we added more debug information to see with microsecond resolution when a message was received and all of the other block propagation processes happened.

We collected all of those debug logs and did some analysis. The data we have got so far, I finished about yesterday at 12 pm, so there's a lot of tests that we want to do but haven't done yet. Right now we have mining data using our London server. Everything that you are going to see is coming out of a London broadcast server. I hope to have a switchover to a Beijing server soon, but I haven't done that yet. Maybe in a few days to see some more various data.

While we were doing these tests, we ran into a few bugs with our data. One of which was that bitcoin-xt has a bug where an unreleased version, only in the master branch, we were using the master branch, we fixed this about 2 or 3 days ago. Before this, it is about 3x slower. Also there was a performance issue about bitcoin core and bitcoin-xt where, there are two variants, it's due to locking. The getblocktemplate code blocks cs main and also blocks cs main, so if a node is mining, then all the code that blocks cs node can't getblocktemplate, in some cases it could take 5 seconds to 30 seconds for getblocktemplate, which is a huge amount of latency. I'll tell you about one of those issues in a few seconds.

Here are the nodes that we had. We had two machines that use Intel atom CPUs. We were giving them 9 MB blocks and seeing how well they did, hint not very well. Here's how block propagation message works. There's an "inv" message which says that Alice has a new block in its inventory. That's about 20 peers. Usually about 8 outgoing connections, then maybe some number of incoming connections. She broadcasts that message to everyone she knows all at once, and then Bob will look and see whether he has that block already. If he doesn't, he asks for the headers, and then immediately afterwards he asks for the blockdata. One of the problems we found is that the getheaders function locks cs main. You can't submit headers, you can't propagate the blocks until getblocktemplate finishes. One quick hack is getdata, get headers, get data. Send the data first, send headers, then send data again to handle a side issue. So Bob says getdata usually, then Alice says here's the block, and then after that, after Bob receives the actual download, he still has to verify it. Then when it completes, it finishes with an update tip message.

We monitored inv message arrival at Bob's computer. We ignored the getdata request, although we did collect it. We measured when the block message arrives after downloading. Also we measured when the block was added to the blockchain for Bob's machine. We also collected some other data that we haven't analyzed yet.

On a macro scale, block propagation looks like this. Each computer that gets this data and uploads to several other computers. This is usually 20 machines in parallel. This means that no machine gets the block until the machine is finished uploading to 20 peers at the same time. At 1 megabyte/sec, it takes 20 seconds until anyone has the block. You will see this line down here where the transmission time and verification time in our data.

Now I am going to move over and give you a quick hint about the severity of the China situation. These are some of my nodes in China. This one is Beijing, Shanghai, Shenzhen and Hong Kong. And my nodes below are my European nodes. I am going to download an 8 MB file from all of these nodes at the same time. We will see if this happens at the same way from... European node is done, last one is finished now, you can see now that Beijing is downloading at 200 kilobytes/sec, and Shanghai is downloading at 50 kilobytes/sec. A few days ago, Shanghai was faster than everywhere. A few days ago, Beijing was fastest. This changes day-to-day, sometimes hour-to-hour. If a node is definitely downloading at 30-40 kilobytes/sec, you are going to have problems. And we did have problems. But only with the Chinese nodes.

So here's our tool. Oops, I should mention do not visit the url I just showed. Please don't hug this to death. Please hold off on visiting this for now.

Here's a list of the 50th percentile time for block transmission for each one of these machines. Some of these have an extra dot, this one is using my current sim block patch, it doesn't seem to help that much. Most of them don't have that. The first dot is when it receives the inv, second one is when finished downloading, and third dot is when finishing updating tip of the blockchain to include that block. You'll see that the times for most of these European and North American and non-Chinese Asian nodes are 20 seconds, it's kinda high but not disastrous. If you go down to China and one of our underpowered Intel atom CPUs, you will see that things get really bad.

Over here you have a scatterplot; some of you might have seen the data that gmaxwell collected about stratum times. You see a block propagation time of about 10 seconds for a 1 MB block. With our data, you see about the same. About 10 seconds for a 1 MB block on average. You can also see that there are at different routes, the orange stuff with a steeper slope and then you have the green stuff with a shallower slope, that's an indication of the distance mostly from the mining node, in this case London. The orange is Europe and North America. You also see a lot of scattered off dark in the various areas. And then there's, most of this data came when we only had one machine that wasn't one hop away from the miner node. That one machine on the second hop is actualy this blue down here. If we go to time since that node saw the block, rather than time since mined, we see the blue is dominating at very fast, it's downloading from a machine that is not overloaded.

Our data looks to be similar to what people see on mainnet for propagation through the whole network. We are seeing this over one hop, though. The main effect is that,... the relay network is really huge. I set these tests up in about 2 weeks on people from off the internet. Things aren't optimized. We have Bitcoin Core nodes that are downloading our blocks but not uploading blocks, so they aren't doing anything helpful. That is why we see some dissimilarities with the gmaxwell latency data.

Once you get around to it, you can play around with this, there's a lot of options here, the ones at the top we don't have data for all the machines. These are multi-gigabyte log files. This gray line by the way is one of my Beijing nodes. Over here we have the Intel Atom CPU, it downloads the block quickly, but it takes about 70 seconds to validate because the CPU is insufficient. This node is in Germany, it retrieved a block from a second node, a nearby node, not a mining node, because the mining node didn't have many connections. The germany node downloaded in about 6 seconds, and took 22 seconds to validate. This would go a lot faster once libsecp256k1 gets in there. These validation times could be addressed by the way that blocks are propagated.

We could propagate unvalidated blocks by a certain bit, in a way that SPV wallets wouldn't mistake them for validated blocks. Sometimes these graphs show- that's traffic for that node. It's often very short and peaky. These machines tend to have pretty good internet connectivity, and the algorithm we're using for uploading the blocks, isn't bandwidth-efficient. It uploads to all 8 or 20 peers or whatever, and after that, the bandwidth is wasted, but if you increase the number of peers you increase the total time but you use more bandwidth. We don't need to have this optimization problem.

We can see when the data was uploaded to a peer and when it started uploading data to the next peer. There's quite a big gap there, and that's the validation delay. We could use that time when the peer has the data but hasn't started uploading to anyone else because they haven't finished downloading and uploading. The Chinese nodes show all kinds of crazy stuff going on, it took 166 seconds to download the whole block, because bandwidth was very poor. Here's a block, most of the traffic is afterwards. This was the bug in pin blocks that we need to track down.

Most of the rest of the world can handle 2 MB, 4 MB, 8 and 9 MB is starting to push it, but I think we can do it. China though cannot. And.. where did it go? ergh. We can do all sorts of stuff. We can start uploading in multiple directions once we get the first byte, we could make it like bittorrent and use DHTs. We need to get the Chinese pools to move their block creation out of the country. They can stick the coinbase transaction into the header in China, but everything else can be done outside of China. This would only require 1 kilobyte of traffic through the Great Firewall, without increasing or decreasing block size.

I will be in China for the next 20 days or so. I will also be working on implementing block torrent (bittorrent stuff). Here are some people who helped, thanks.

Q: I will try to be brief. I am skeptical about increasing the block size to 8 MB. In Bitcoin network, .. propagation through 12 nodes, if you have 1 MB block size, and 20 seconds to propagate. If increasing to 8 MB, it means 120. If you are multiplying it, so it's meaning it will be at least 5 or 10 minutes delay even in shortest. The next want, if we are talking about Bitcoin blockchain increasing to 8 MB, meaning 500 GB per year in blockchain, ... I will just ... my question is, how you would like to solve this in bitcoin-xt?

A: My talk is just about block propagation, I don't have time to address other questions.

Q: What you have shown is that it is only 10s of nodes. But in real life you need at least 500 nodes.

A: We don't have the relay network. That's one difference between us and mainnet. The node count is another difference between us and mainnet. The relay network takes a much shorter path between mining nodes. The number of hops decreases. The size of data sent also decreases. What I was trying to get at for gmaxwell data for adjusting for large opposing effect, our block propagation data for 1 MB propagation data, mostly agrees with mainnet propagation data. It's not the perfect answer, but that's the data that I have.

Q: For big block proposals, what about storage hardware costs for running a full node? How will this develop over time?

A: My two quick answers for that are, checkpoints and pruning. Those can reduce the storage requirements for the blockchain. There will be people who want to store the whole blockchain and they will make it archival and available for the general public. We have quite a while until 8 GB blocks. I don't know how ferromagnetic RAM is going to progress. I think it will be cheap to run a fully-validating node or miner, without full history, will just require the UTXO set. We have some of this in Bitcoin Core, it's just not quite done. We don't have to store the entire history at every single node.

Q: Issues with raising the block size, worst case scenario?

A: We ran into some bugs for locking issues. Performance issues in Bitcoin Core and bitcoin-xt. One of the performance issues was specific to bitcoin-xt. The main serious issue that we have is the issue that petertodd talked about, which is competitive access to blocks in China, which is not good for decentralization. What could happen? Bitcoin could start a nuclear war somehow, not sure.

Q: How big are the UTXO set size were you testing with?

A: The blockchain was 3.3 GB when we started. I didn't look at the UTXO set size in particular. Sorry, I can't answer that question. I can look into it.

Q: It seemed like your blocks were transmitting always from Lond. What about data from when blocks were generated in China? I am concerned about the internet issue is something that circles China. 

A: I don't have data from mining in China yet. However, I can do a similar test, which I actually prepared for, where I downloaded the file from a Chinese node. Pick a city. Okay, Shenzhen. This isn't here, this is all in my notes. Oh, we lost the screen. Shanghai is getting packet loss from the Great Firewall. Washington had 512 kilobytes/sec. Tokyo 812 kilobytes/second. Singapore still waiting, hasn't even started now. 980 kilobytes for London. We were using teamviewer to show my laptop.

Q: Relay network and block compression. Maybe it's more realistic to test without that, because it's a lot closer to an adversarial situation.

A: That's one of the reasons why I didn't include it. It was mostly lazyness and lack of time. But I wanted to figure out situations in which it could break. We did break it a couple of times, but not in ways we were not expecting. We hard-forked testnet 6 times, and we mined for about a day on slightly less hashpower on the bip101 fork, so we were orphaning every block, which is expected behavior for how hard-forks work.