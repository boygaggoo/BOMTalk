# BOMTalk

A rather unknown part of [GameKit](http://bit.ly/iCr99E) is its ad hoc networking between devices by means of Bluetooth or WiFi using Bonjour services. This immensly simplifies sharing data between iOS devices, but the API is unconvenient for most purposes and somehow aging, offering no clean delegate or block based callback mechanism.

This is what BOMTalk offers:

- GameKit based with support for Bluetooth and WiFi ad hoc networks
- provides either a block-based or delegate-protocol or (multicast) notifications API
- sends and receives arbitrarily sized data (NSData conforming to NSCoding)
- automatically keeps one connection slot open for up to 15 simultaneous connections per session
- hides GameKits weird delegates/class callbacks and consolidates its states
- basic network debugging view controller to analyse communication

# Examples

## Pastboard

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/Pasteboard.png">
[Pasteboard.png](https://raw.github.com/omichde/BOMTalk/master/Sample/Pasteboard.png)

This example sends your current pasteboard data (image or text) to another device of your choice. The receiving device will display the image or text and copies it to its own pasteboard.

## Roll the Dice

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/RollTheDice.png">
[RollTheDice.png](https://raw.github.com/omichde/BOMTalk/master/Sample/RollTheDice.png)

All devices will roll a random number and the one who wins will be displayed in green, the others will see a red number.

## Pong

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/Pong.png">
[Pong.png](https://raw.github.com/omichde/BOMTalk/master/Sample/Pong.png)

This rough mini-game connects two player and lets them play pong against each other. With only basic UI and highscores this gives a good start for fast communication between devices in BOMTalk.

## Remote Camera

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/RemoteCamera.png">
[RemoteCamera.png](https://raw.github.com/omichde/BOMTalk/master/Sample/RemoteCamera.png)

Taking a remote photo from a different iPhone can be useful (e.g. recursive photos or self-portraits without *the* arm), this APP allows you to request a photo from another iPhone in maximum quality to be sent to your device and saved to the photo album. The idea was born at [UIKonf](http://www.uikonf.com) by two attendees.

# Concepts

As BOMTalk sits atop of [GameKit](http://bit.ly/iCr99E), you should probably become familiar with [GameKit](http://bit.ly/iCr99E)'s concepts first. In contrast to GK and to minimize teh API, states and methods, BOMTalk auto-connects to all visible peers in GKSessionPeerMode.

BOMTalk needs only two classes to interact with: BOMTalk and to a lesser extend BOMTalkPeer. All you have to do is prepare your data to be NSCoding compatible for transmission and a little "protocol":

## Your Protocol

Although BOMTalk is built to let your APPs "talk" to each other, you need to define the "language" or the "protocol" to achieve this interaction. This protocol is sometimes described by [DAGs](http://bit.ly/5UhB), a neat way to visualize the different states/devices and the connections between them. Basically you draw circles for the states/devices and connect them with arrows. These arrows are the messages (with optional payload/data) which are sent between devices/APPs. In BOMTalk each messages is a number (with optional payload/data) and could be as simpel as this one-direction protocol, sending one photo directly to another device:

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/dag.png">
[dag.png](https://raw.github.com/omichde/BOMTalk/master/Sample/dag.png)

Depending on your APP logic and needs you most probably add much more messages to your APP. For example, *requesting* a photo from another device requires two messages:

<img src="https://raw.github.com/omichde/BOMTalk/master/Sample/dag-double.png">
[dag-double.png](https://raw.github.com/omichde/BOMTalk/master/Sample/dag-double.png)

Testing your "protocol" can be an extremely tedious task: modelling states in APPs takes some time to figure out correctly, network connections are unreliable, they are delayed or will never arrive at all. For that reason, BOMTalk offers a basic network messaging debugger BOMTalkDebugViewController (on top of GameKit) with a timeline of appearing and disappearing peers and events/messages between those peers.

## BOMTalk, BOMTalkPeer, BOMTalkPackage

[BOMTalk]([BOMTalk]) is the primary class your controller interacts with. It's meant to be used as a singleton and encapsulates all networking and objects. Each peer, which BOMTalk connects and talks to, is encapsulated in a BOMTalkPeer class. If you like to attach additional data to a peer, use its [userInfo]([BOMTalkPeer userInfo]) dictionary for that purpose. Internally BOMTalkPackage is used to transfer data between peers but that is the least interessing class for you (generally you interact with BOMTalk 95% and 5% with BOMTalkPeer).

## Data and Progress

Your data must adhere to the NSCoding protocol!

Apple's size recommendation to send data through GameKit is max. 50KB, but BOMTalk allows arbitrary sized NSData. In the sender BOMTalk splits the data into blocks, collects them on the receivers end and recreates the previously NSData object.

Sending data is completely optional: in a lot of cases, message IDs may suffice (for states, handshaking etc). If you send data and it exceeds the limit, you can be notified of the transmission progress for each block (not for individual bytes).

# Documentation

BOMTalk consolidates GameKits APIs and internal techniques by providing a BOMTalk connection "singleton". This object ecapsulates the network handling, keeps its list of peers up to date and interacts with your controller in three ways: blocks, delegates or notifications. Alongside the obvious [sendTo]([BOMTalk sendToAllMessage:])-messages all callback mechanisms are modelled with the same concept in mind:

- answering to incoming messages
- progess metrics for outgoing and incoming data
- general network updates

## 1. Blocks

Blocks immensly simplify your code logic in your controller. As an example (excerpt from the Remote Camera example) for sending an image between two devices, only a few lines of code are needed to send or store the image:

	// SENDER
	BOMTalk *talker = [BOMTalk sharedTalk];
	[talker start];
	[talker sendMessage:kSendPhoto toPeer: peer withData: UIImageJPEGRepresentation(image, 1)];

	// RECEIVER
	BOMTalk *talker = [BOMTalk sharedTalk];
	[talker start];
	[talker answerToMessage:kSendPhoto block:^(BOMTalkPeer *peer, id data) {
		UIImage *image = [[UIImage alloc] initWithData: data];
		UIImageWriteToSavedPhotosAlbum(image, nil, nil, nil);
	}];

## 2. Delegates

If you need a seperate object to react to your network messages or prefer the common delegate mechanism, you can use it with BOMTalk as well. As another example (excerpt from the Roll the Dice example) to answer to a remote request, your APP receives the request and sends data back like this:

	// SENDER
	- (void) viewDidLoad {
		BOMTalk *talker = [BOMTalk sharedTalk];
		talker.delegate = self;
		[talker start];
	}

	- (IBAction) roll {
		// select peer, then
		[talker sendMessage:kRollStart toPeer: peer];
	}

	// RECEIVER
	- (void) viewDidLoad {
		BOMTalk *talker = [BOMTalk sharedTalk];
		talker.delegate = self;
		[talker start];
	}

	- (void) talkReceived:(NSInteger) messageID fromPeer:(BOMTalkPeer*) sender withData:(id<NSCoding>) data {
		switch (messageID) {
			case kRollStart:
				[[BOMTalk sharedTalk] sendMessage:kRollAnswer toPeer:sender withData: [NSNumber numberWithInt: random() % 100]];
			break;
		}
	}

## 3. Notifications

If you need to handle messages in multiple objects (multicasting), you can use iOS notifications: add an observer to the BOMTalkNotifications with your methods to be called. As an example (excerpt form the Pong example) if you want to send data from one device to another, here is an example with notifications:

	// SENDER
	- (void) viewDidLoad {
		[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(connectedTalk:) name:BOMTalkDidConnectNotification object:nil];
		BOMTalk *talker = [BOMTalk sharedTalk];
		[talker start];
		// select peer, then:
		[[BOMTalk sharedTalk] sendMessage:kGameBall toPeer:peer withData: @{@"x": [NSNumber numberWithFloat:pos.x], @"y": [NSNumber numberWithFloat:pos.y]}];
	}

	// RECEIVER
	- (void) viewDidLoad {
		[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(receivedTalk:) name:BOMTalkReceivedNotification object:nil];
		BOMTalk *talker = [BOMTalk sharedTalk];
		[talker start];
	}
	- (void) receivedTalk: (NSNotification*) notification {
		BOMTalkPeer *peer = notification.object[@"peer"];
		NSNumber *message = notification.object[@"messageID"];
		switch (message.integerValue) {
			case kGameBall:
				_ballView.center = CGPointMake([notification.object[@"data"][@"x"] floatValue], [notification.object[@"data"][@"y"] floatValue]); 
			break;
		}
	}

For notifications, all additional data is sent in the `object` parameter as follows:

- *BOMTalkReceivedNotification*
	NSDictionary with keys:
	`messageID` - NSNumber with then message ID
	`peer` - BOMTalkPeer with the sending peer
	`data` - id<NSCoding> (optional)

- *BOMTalkProgressForReceivingNotification*
	NSNumber with float [0.0 .. 1.0]

- *BOMTalkProgressForSendingNotification*
	NSNumber with float [0.0 .. 1.0]

- *BOMTalkUpdateNotification*
	BOMTalkPeer of the peer

- *BOMTalkFailedNotification*
	NSError describing the error

# Installation

Copy the topmost `BOMTalk` folder to your project, add the `GameKit.framework` to your project and include the `BOMTalk.h` header file. Maybe I'll split the project later on to move the samples into another repository.

# Version

**1.0**

- production ready ([Exhibition's](http://exhibition.werk01.de) file sharing and remote control feature will use BOMTalk)

**0.93**

- minimized API, fixed samples
- peerList now contains connected peers

**0.92**

- dropped asServer property
- fixed over-sorted peerList array

**0.91**

- block, degelegates and notifications are *not* mutually exclusive anymore

**0.9:**

- a new XCode build target creates an [appledoc](http://gentlebytes.com/appledoc/) documentation from the sources
- documentation to illustrate BOMTalk in practice

**0.8:**

- basic network debug view controller
- multiplayer pong basically working
- new delegate sample: remote camera

**0.7:**

- prepared pong game for multiplayer

**0.6:**

- new sample: pong game (local mode only)

**0.5:**

- streamlined API for blocks, delegates and notifications

**0.4:**

- progress API

**0.3:**

- merged some callbacks
- refined API

**0.2:**

- expanded delegates

**0.1:**

- two samples: Pasteboard, Roll-the-Dice

# Contact

Oliver Michalak - [oliver@werk01.de](mailto:oliver@werk01.de) - [@omichde](http://twitter.com/omichde)

## Keywords

iOS, GameKit, Bluetooth, WiFi, ad hoc, network, blocks, delegate, multicast

## License

BOMTalk is available under the MIT license:

	Permission is hereby granted, free of charge, to any person obtaining a copy
	of this software and associated documentation files (the "Software"), to deal
	in the Software without restriction, including without limitation the rights
	to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
	copies of the Software, and to permit persons to whom the Software is
	furnished to do so, subject to the following conditions:

	The above copyright notice and this permission notice shall be included in
	all copies or substantial portions of the Software.

	THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
	IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
	FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
	AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
	LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
	OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
	THE SOFTWARE.
