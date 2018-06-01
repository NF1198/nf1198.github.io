---
layout: post
title:  "NmapToCSV"
date:   2018-06-01 13:40:00 -0600
categories: Nmap Java XML
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

[Nmap](https://nmap.org/) is a handy tool for network and service discovery. It
allows you to scan entire networks and identify rogue hosts and can often yield
unexpected results. 

By default, Nmap stores scan results in an XML format and doesn't offer any post-processing
tools. You're out-of-luck if you need to quickly process a batch of scan results or
merge datasets for analysis.

# NmapToCSV

[NmapToCSV](https://github.com/NF1198/NmapToCSV) is a lightweight command-line 
utility for post processing Nmap XML output.

For example, given a directory of XML scan results, you can run:

    $> nmaptocsv exportHosts -D <path to NMAP XML> | column -s, -t
    exportHosts
    IPv4           hostname  service           port   proto  state   product
    192.168.1.1              domain            53     tcp    open    dnsmasq
    192.168.1.1              http              80     tcp    open    GoAhead WebServer
    192.168.1.1              pptp              1723   tcp    open
    192.168.1.115            ssh               22     tcp    open    OpenSSH
    192.168.1.115            http              80     tcp    open    Apache httpd
    192.168.1.115            netbios-ssn       139    tcp    open    Samba smbd
    192.168.1.115            netbios-ssn       445    tcp    open    Samba smbd
    192.168.1.185            msrpc             135    tcp    open    Microsoft Windows RPC
    192.168.1.185            netbios-ssn       139    tcp    open    Microsoft Windows netbios-ssn
    192.168.1.185            microsoft-ds      445    tcp    open
    192.168.1.185            http              5357   tcp    open    Microsoft HTTPAPI httpd
    192.168.1.214            unknown           6646   tcp    closed
    192.168.1.214            realserver        7070   tcp    closed
    192.168.1.214            http-alt          8000   tcp    closed
    192.168.1.214            http              8008   tcp    closed
    192.168.1.214            ajp13             8009   tcp    closed
    192.168.1.214            http-proxy        8080   tcp    closed
    192.168.1.214            blackice-icecap   8081   tcp    closed
    192.168.1.214            https-alt         8443   tcp    closed
    192.168.1.214            sun-answerbook    8888   tcp    closed
    192.168.1.214            jetdirect         9100   tcp    closed
    192.168.1.214            abyss             9999   tcp    closed
    192.168.1.214            snet-sensor-mgmt  10000  tcp    closed
    192.168.1.33             ssh               22     tcp    open    OpenSSH
    192.168.1.33             http              80     tcp    open    Apache httpd


Checkout the [github project](https://github.com/NF1198/NmapToCSV) for example usage.