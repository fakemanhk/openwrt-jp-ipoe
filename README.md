# **Configuring OpenWRT to work with Japan NTT IPv6 (MAP-E) service**

## **The problem:**

ISPs with NTT mostly support both IPv4 & IPv6 implementations, while former one usually by using PPPoE which can introduce higher latency, during peak hours it can be also very slow in some busy districts. IPv6 is their newly promoted way to connect to internet which doesn't require PPPoE,  they also claim this is a much faster option, with IPv4 over IPv6 together users should retain traditional IPv4 connectivity. Unfortunately if you subscribe the internet service without using Hikari Denwa (ひかり電話) residential phone service, you will end up getting /64 prefix address as well as without router advertisement (RA), if you don't use vendor provided router it would be extremely difficult to set up your IPv6 network with IPv4 over IPv6 connectivity.

There exist a few different implementations (DS-LITE/Transix/MAP-E)  in Japan, so not all providers can do the same way, here I am only referring to my own provider _**NTT ぷらら (plala**_**),** which is MAP-E implementation.

## **Setup**:

The followingFollowig setup was tested with **GL-INET MT1300 (beryl)** and **NanoPi R4S (4GB)** installing vanilla OpenWRT v22.03 firmware

*   **System > Software**: Install the required add-on package _**map**_ for MAP-E/MAP-T support, you will need to reboot before you can use it.
*   Enable WAN6 with **DHCPv6**, firewall setting you probably need to add this to **WAN Zone** (same as IPv4 WAN) for protection.

> Under **DHCP Server > IPv6** Setting, follow these settings:
> 
> *   Designated master ON
> *   RA-Service: relay mode
> *   DHCPv6-Service: relay mode
> *   NDP-Proxy: relay mode
> *   Learn routes: ON

![](https://user-images.githubusercontent.com/21307353/212850790-a2c4ac8b-3bed-4941-8f1a-49e9e5597c3f.png)

Save & Apply setting, you should see a public IPv6 address being assigned to your _WAN6_ interface (usually starting with 2400)

*   _WAN6_ interface also needs to add _**Customized Prefix Delegation**_, using SSH to login router, edit **/etc/config/network,** add the line marked in _**Italics**_ under _WAN6_ interface section, note the _**2400:aaaa:bbbb:cccc**_ is your WAN IP prefix (first 64 bit), this will give the WAN6 interface proper IPv6-PD:

> config interface 'wan6'
> 
>             option device 'eth1'
> 
>             option proto 'dhcpv6'
> 
>             option reqaddress 'try'
> 
>             option reqprefix 'auto'
> 
>             _**option ip6prefix '2400:aaaa:bbbb:cccc::/64'**_

Note: Previously I had failed my setup because of missing this step, it wasn't mentioned in most resources I found on web, and I eventually got a _**MAP rule invalid**_ error.

*   Next, configure LAN interface, under **DHCP Server > IPv6** settings, basically very similar to WAN6 but **Designated master OFF**

![](https://user-images.githubusercontent.com/21307353/212852863-7fb85f4e-a04c-4ed6-b936-2b71f2631019.png)

You clients can probably get public IPv6 addresses from router now! But this will be IPv6 access only and you are still missing the IPv4 connectivity, another MAP-E interface is required to fill in the gap.

*   Before we create the MAP-E interface, the parameter calculation for MAP-E is needed, in reference section I have attached the IETF information but someone has created a page to calculate, you can copy the public IPv6 address from WAN6 interface and use this [online MAP-E rule calculator](http://ipv4.web.fc2.com/map-e.html):

![](https://user-images.githubusercontent.com/21307353/212853420-6ce2090f-98f1-4f34-9f44-4db2d3bbddca.png)

Note: Before I ran the above calculator, I used the Buffalo router that came with ISP to connect the internet service, logged into that router and from status page I can see that at least the IPv4 address and port numbers are the same as above, so I believe the parameters I get from the calculator should be correct, you might want to do this as a verification.

*   Next will be setting up new MAP-E interface, create a new interface and name it (e.g. _WAN6MAPE_), and fill the parameters using above generated parameters:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: _\[peeraddr\]_
    *   IPv4 prefix: _\[ipaddr\]_
    *   IPv4 prefix length: _\[ip4prefixlen\]_
    *   IPv6 prefix: _\[ip6prefix\]_
    *   IPv6 prefix length: _\[ip6prefixlen\]_
    *   EA-bit length: _\[ealen\]_
    *   PSID-bits length: _\[psidlen\]_
    *   PSID offset: _\[offset\]_
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**:

![](https://user-images.githubusercontent.com/21307353/212856884-d6d627a4-37b9-4002-99a7-2795dccac2cd.png)

Note: Don't forget to add this _WAN6MAPE_ interface to same firewall zone as WAN/WAN6 since this is also part of WAN.

Eventually you should see the following screen under **Network > Interfaces**, _WAN6MAPE_ should get the IPv4 exactly the same as using the ISP provided router, there is also a _**Virtual dynamic interface**_ automatically created when MAP-E interface started correctly.

![](https://user-images.githubusercontent.com/21307353/212857199-21f283c9-e9e2-43b2-8d58-955126076744.png)

From **Status > Overview** you'll see both **IPv4 Upstream** and **IPv6 Upstream** information:

![](https://user-images.githubusercontent.com/21307353/212858791-e21a621e-0a5a-40a9-952f-ec9d759b6a9e.png)

Testing with my Linux laptop by visiting the [OCN connectivity verification](https://v6test.ocn.ne.jp/) page, both IPv4/IPv6 addresses should be the same as above upstream informations:

![](https://user-images.githubusercontent.com/21307353/212859123-0590650f-29f1-412c-99a7-19d5516a8d22.png)

## **SUCCESS!!**

Note: Some clients might have issues with DHCPv6, you can refer to the discussion [here](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/2).

Things to follow up later: For most 1G internet package PPPoE (IPv4 only) and IPoE (IPv6 with v4 compatibiliy) can usually coexist (10G plan should have no PPPoE now), meaning that you can connect ISP ONU to a switch, with one port connecting with IPoE, and the other one with traditional PPPoE. The PPPoE is still useful here in case you need to open server at home, might try later to see if I can add another virtual interface to WAN side for PPPoE dialup.

Reference sites:

https://datatracker.ietf.org/doc/html/draft-ietf-softwire-map-03#page-6

[https://www.labohyt.net/blog/lan/post-6760/](https://www.labohyt.net/blog/lan/post-6760/)

[https://zenn.dev/yakumo/articles/19cbc6309d8143cc9349b2fb0d29771e](https://zenn.dev/yakumo/articles/19cbc6309d8143cc9349b2fb0d29771e)

[https://blog.hinaloe.net/2020/03/14/openwrt-mape-ocn/](https://blog.hinaloe.net/2020/03/14/openwrt-mape-ocn/)
