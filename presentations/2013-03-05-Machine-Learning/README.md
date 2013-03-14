# Video Overview

This presentation gives a technical overview of the Privly system. It is under development with the intention of recording it for the general public and posting it to the web.

## Presentation Outline
---
---
# Problem Statement
* Countries without strong press-freedoms regularly censor content.
* Countries and individuals often compromise OSP user accounts and gain access to a trove of unencrypted personal communications.
* OSPs subject content to continually changing permission structures and system capabilities, which results in accidental leaks of information.
* Even sophisticated web technology companies have security vulnerabilities that compromise billions of user photos and messages.
* Content posted to OSPs is often jointly owned by the OSP, which can destroy the content's value.

# Failures of Current Solutions

## Specificity

Talk about Gdocs

Solution: Adapt to multiple use cases and web applications

## Apathy

Talk about PGP (give xkcd graphic)

Solution: Add user value beyond privacy and security

## Bootstrapping

Talk about Diaspora

Solution: Fail gracefully by working without the need of any specialized client code

# Introducing Privly

Show the Twitter Example

# Threat Model

The Privly threat model considers OSPs to potentially be adversarial. OSPs have read and write access to all user data. In community sites, the OSP may impersonate users, change user content, destroy user content, or leak data to third parties. Furthermore, OSPs can remove, restrict access to, or modify content once posted. We also consider governments and ill-willed users to potentially be adversarial, and capable of either colluding with the OSP or exploiting vulnerabilities in the OSP itself.

We assume that the browser has access to a secure transport protocol in all communications. Further, we assume that the user's browser is not compromised and that all servers storing content have not been compromised.

# Privly's Inspiration

CaaS and Scramble!

## CaaS

Trust Separation

## Scramble!

Public Key Cryptography

## How Do We Combine the Two?

### GreaseMonkey of "Injectable Applications"

From an architectural perspective, Privly can be viewed as a Greasemonkey [39] for web security ap- plications.	The Greasemonkey browser extension is a framework upon which the browsing experience is selectively altered by installing community-contributed scripts. Such scripts are often developed to increase the accessibility of websites [9] or modify site functional- ity. The Greasemonkey installation base is nearly three million users [36, 28], and several privacy applications use Greasemonkey for their proof of concept implemen- tations [21, 19]. Privly brings Greasemonkey’s approach of script development and management to the application level.

## Twitter Posting Picture

## Twitter Reading Picture

# Components

## Injectable Application

(Twitter Reading Picture)

## Injectable Application

### What Can These Applications Do?

* Full web applications: Cryptography, Chat, Email, Code Sharing
* With the added benefits: Work with any web server, integrates with existing websites, can customize APIs for host sites, 


## Extension

Posting Form

## Host Page

Interact with imo.im

## Content Server

Privly or anyone else, show Github

## Library

Provided by the extension and a content server

# Posting Diagram

# Reading Diagram

# Analysis

## Spoofing

(Spoofing screen capture)

## Tracking

## Host-P knows content length

## User Understanding

## Distributed Denial of Service

## Privly Blocking

## Other Extensions

# Current Implementation

## Content Server

## Extensions Table

