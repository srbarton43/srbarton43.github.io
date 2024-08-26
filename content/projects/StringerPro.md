---
title: StringerPro
author: Samuel Barton
date: "2023-04-22"
summary: A companion app for racquet stringers
description: A helpful native application for racquet stringers written in python
tags: ["application", "software", "python"]
cover:
    image: /projects/stringerpro/cover.png
    caption: "Clickable list view for a selected brand"
    hiddenInList: true
TocOpen: false
ShowToc: true
---

## About

Since I string tennis racquets quite often as a part time job, I must check online various racquet-specific information such as the string pattern, holes to skip, and string length required.
This information is usually hard to find, and the one website I know of, provides the data in an illegible table.
Therefore I created this project to allow myself (and other users) to easily access racquet-specific string pattern information.

## Source Code

[Link to Repository](https://github.com/srbarton43/StringerPro "git repo")


## Design

Use of the GUI: The user can select the racquet brand from the top drop-down menu. Then a list of clickable racquet models appears, or a user can search for a specific model which can also be clicked in the list to display its string pattern specifications. It is quite intuitive to use (although old looking). Since PyQt is cross-platform, this app runs on Mac OS, Windows, and Linux distributions.

{{< figure src="/projects/stringerpro/extreme.png" caption="View of a specific racquet model" >}}

## Implementation

This entire project is written in python, the programming language I am currently using the most during my Spring 2023 Software Internship. The two main libraries which this project uses are BeautifulSoup4 for web scraping, and PyQt6 for the cross-platform graphical user interface. When the app starts up, it immediately starts scraping the home directory of www.klipperusa.com/pages/racquet-stringing-patterns to access its linked webpages for each racquet brand. For each brand the program creates a hash table of [Model->Specifications] key->value pairs so that this information will not have to be dynamically generated as the user interacts with the app. 
