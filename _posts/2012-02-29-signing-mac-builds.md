---
title: Signing Mac Builds
author: edransch
layout: post
permalink: /2012/02/signing-mac-builds/
categories:
  - Mozilla
  - Releng
tags:
  - apple
  - mozilla
  - releng
  - signing
---
For the past month or so I have been working with [Ben][1] to add functionality for the signing of mac builds to our current signing infrastructure. Along the way, we discovered a few things.

## What is signing?

Code signing is the process of attaching a digital signature to a piece of software that allows the user&#8217;s OS to verify that the software does indeed come from its advertised source, and that it has not been altered since being signed.

## Why do we need signing?

Apple&#8217;s recently announced Mountain Lion is going to be introducing a few new features. In particular the new Gatekeeper ( [Learn more here][2] ) is important for Release Engineering at Mozilla. The default setting for Gatekeeper will \*only\* allow Applications to be installed from The App Store or from Registered Developers. This means that any applications are not signed by a Registered Apple Developer will show Security warnings when users install them. To ensure that our users have a seamless experience, we need to make sure that our builds are signed correctly according to Apple&#8217;s standards.  
[<img class="aligncenter" src="http://images.apple.com/macosx/mountain-lion/images/security_settings.jpg" alt="" width="412" height="354" />][2]  
Gatekeeper isn&#8217;t the only reason Code signing is important. Signing verifies that the contents of the Application a) come from the trusted developer and b) have not been altered. OSX also uses the code signatures to determine whether an Application is trustworthy enough to be allowed Keychain access.

## Code Signing on OSX

Apple provides a tool for Signing Applications called &#8216;codesign&#8217;. The &#8216;codesign&#8217; tool will apply a digital signature to an entire &#8216;.app&#8217; directory.

There are two parts to the signature:

  1. A generated manifest containing hashes of each file in the directory. This file is located in &#8216;*.app/Contents/_CodeSigning/CodeResources&#8217; and is generated according to a specified rules file (more on this later).
  2. A digital signature attached to the binary specified as &#8216;CFBundleExecutable&#8217; in Info.plist . This signature also contains the hash of the generated CodeResources file.

&#8216;Codesign&#8217; will verify a given &#8216;.app&#8217;s signature with the -v option. (Be sure to add a second v to get &#8216;verbose&#8217; output, otherwise the only indication of success is the return code)

<pre class="brush: plain; light: true; title: ; notranslate" title="">$ codesign -vv Firefox.app
Firefox.app: valid on disk
Firefox.app: satisfies its Designated Requirement
</pre>

## Automating Signing

There are a few important details about code signing on OSX which were important to address.

### OS Compatibility

Applications which are signed on OS 10.7 (Lion) do not verify on 10.5 (Leopard). Since we want to be running one set of servers to sign all of our builds, we need signed builds to verify on 10.5, 10.6, and 10.7. Luckily, applications signed on OS 10.6 (Snow Leopard) verify correctly on all three versions. We&#8217;ll be using 10.6 as the OS on our Mac signing servers at least until 10.8 (Mountain Lion) is released, at which point we&#8217;ll need to re-evaluate.

&nbsp;

### Keychain Popup

Apple&#8217;s &#8216;codesign&#8217; command require an signing key, or ID, which is stored in a Keychain. The ID and Keychain are passed as arguments to &#8216;codesign&#8217;. Unfortunately, &#8216;codesign&#8217; does not offer any way to enter the password at the command line. If it tries to access a locked Keychain, it will pop up a UI prompt, asking the user for the Keychain&#8217;s password.

<div id="attachment_117" style="width: 472px" class="wp-caption aligncenter">
  <a href="http://www.erickdransch.com/blog/wp-content/uploads/2012/02/KeychainPop.jpg"><img class="size-full wp-image-117" src="http://www.erickdransch.com/blog/wp-content/uploads/2012/02/KeychainPop.jpg" alt="The bane of release engineers worldwide" width="462" height="242" /></a>
  
  <p class="wp-caption-text">
    The bane of release engineers worldwide
  </p>
</div>

Thankfully for those of us who like to automate things, the &#8216;security&#8217; tool comes to our rescue here. &#8216;Security&#8217; is another Apple tool that allows command-line access and manipulation of Keychains. And &#8216;security&#8217; does allow the Keychain&#8217;s password to be entered either as an argument or in response to a terminal prompt.

<pre class="brush: plain; light: true; title: ; notranslate" title="">$ security unlock-keychain ***.keychain
password to unlock ***.keychain:
</pre>

One important thing to note about the &#8216;security unlock-keychain&#8217; command is that it will only unlock the Keychain for the current security context. In particular this means that if the Keychain has been unlocked by the command being run in a terminal, it is not unlocked for any ssh connections into the machine.  
To user &#8216;security unlock-keychain&#8217; without user interaction, the password needs to be entered at the tty level. [Pexpect][3] is one option for python which will wait for the command-line prompt and enter the password.  
Now we&#8217;ve got all the tools we need to automate signing with the following steps with no user interaction:

  1. Unlock the keychain with &#8216;security unlock-keychain&#8217; and enter the passphrase using pexpect
  2. Run the &#8216;codesign&#8217; command
  3. Lock the keychain again with &#8216;security lock-keychain&#8217;

### Code Resources

The CodeResources file is used by &#8216;codesign&#8217; to specify which files in the &#8216;.app&#8217; directory need to be included in the signature and which files do not. Before signing, the CodeResources file contains a set of rules. After signing, the hashes of all files specified in the rules will be added to the CodeResources file.  
[This][4] page by Apple provides some description and examples of the CodeResources file, but it fails to explicitly state an important fact: any path specified in the CodeResources rules will be interpreted recursively to include all of the files in that directory and below.  
The CodeResources rules must specify all files that need to be included in the final signature. Any files not listed or not in any of the recursive includes will be ignored, and will not have their hashes added to the signed version of the CodeResources file. Any files that are included in a recursive listing of a directory can be explicitly omitted. The file specified as &#8216;CFBundleExecutable&#8217; in Info.plist is never included in the signed CodeResources file. Instead it is signed directly by the &#8216;codesign&#8217; command.

The following example will use this simple directory structure:

<pre class="brush: plain; light: true; title: ; notranslate" title="">Test.app
    Contents
        Info.plist
        MacOS
            binary &lt;-- the binary file listed in Info.plist
            anotherfile
        Resources
            omittedfile
            resourcesfile
</pre>

Before signing, we specify a set of rules in the CodeResources file. These rules will be passed to the &#8216;codesign&#8217; command when it is called. The following code specifies these rules:

  * Recursively include the &#8216;MacOS&#8217; directory and the &#8216;Resources&#8217; directory 
  * Omit &#8216;Resources/omittedfile&#8217; from signing.
  * Include &#8216;version.plist&#8217; in the signature

<pre class="brush: xml; title: ; notranslate" title="">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd"&gt;
&lt;plist version="1.0"&gt;
&lt;dict&gt;
	&lt;dict&gt;
        &lt;!-- Recursively include 'MacOS' and 'Resources' dirs --&gt;
		&lt;key&gt;^MacOS/&lt;/key&gt;
		&lt;true/&gt;
		&lt;key&gt;^Resources/&lt;/key&gt;
		&lt;true/&gt;
        &lt;!-- Omit a file --&gt;
		&lt;key&gt;^Resources/omittedfile$&lt;/key&gt;
		&lt;dict&gt;
			&lt;key&gt;omit&lt;/key&gt;
			&lt;true/&gt;
			&lt;key&gt;weight&lt;/key&gt;
			&lt;real&gt;1100&lt;/real&gt;
		&lt;/dict&gt;
		&lt;key&gt;^version.plist$&lt;/key&gt;
		&lt;true/&gt;
	&lt;/dict&gt;
&lt;/dict&gt;
&lt;/plist&gt;
</pre>

The after signing, the CodeResources file has a list of the hash of every file specified in the rules. If any of these files are changed, &#8216;codesign -vv&#8217; will fail and will specify the changed file. Notice that the omitted file does not have its hash listed here, nor does the Info.plist file, which was not included explicitly, nor in any of the recursive includes. The binary file has been signed separately by &#8216;codesign&#8217; and is not included in this file.

<pre class="brush: xml; title: ; notranslate" title="">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd"&gt;
&lt;plist version="1.0"&gt;
&lt;dict&gt;
	&lt;key&gt;files&lt;/key&gt;
	&lt;dict&gt;
		&lt;key&gt;MacOS/anotherfile&lt;/key&gt;
		&lt;data&gt;
		K2QRL8qPGNuexZXewkmnMlmcdKU=
		&lt;/data&gt;
		&lt;key&gt;Resources/resourcesfile&lt;/key&gt;
		&lt;data&gt;
		PhrNXR4JvoBSBHWVsGOxbosr2po=
		&lt;/data&gt;
	&lt;/dict&gt;
	&lt;key&gt;rules&lt;/key&gt;
	&lt;dict&gt;
		&lt;key&gt;^MacOS/&lt;/key&gt;
		&lt;true/&gt;
		&lt;key&gt;^Resources/&lt;/key&gt;
		&lt;true/&gt;
		&lt;key&gt;^Resources/omittedfile$&lt;/key&gt;
		&lt;dict&gt;
			&lt;key&gt;omit&lt;/key&gt;
			&lt;true/&gt;
			&lt;key&gt;weight&lt;/key&gt;
			&lt;real&gt;1100&lt;/real&gt;
		&lt;/dict&gt;
		&lt;key&gt;^version.plist$&lt;/key&gt;
		&lt;true/&gt;
	&lt;/dict&gt;
&lt;/dict&gt;
&lt;/plist&gt;
</pre>

## That&#8217;s it for now

Look for Mac signing servers in the next few weeks at Mozilla. I hope this helps us provide a more seamless user experience (maybe including Keychain access?), and readies us for the release of OS 10.8 (Mountain Lion).

 [1]: http://blog.mozilla.com/bhearsum/
 [2]: http://www.apple.com/macosx/mountain-lion/security.html
 [3]: http://www.noah.org/wiki/pexpect
 [4]: https://developer.apple.com/library/mac/#technotes/tn2206/_index.html