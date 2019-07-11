---
layout: post
title:  "Program Xilinx Spartan 6 using FTDI FT2232H"
date:   2019-07-11 00:00:00 +0800
categories: fpga
---
I started research on FPGA with a Xilinx Spartan-6 development board. The board was bought 3 years ago from Aliexpress and costs about 30$ which was a good deal back when there were not many alternatives.

{% include image name="fpga.png" caption="Xilinx Spartan-6 Dev Board" %}

The development board came with a parallel cable for programming/flashing the FPGA. Fortunately, my previous desktop is old enough to have a parallel port.

{% include image name="parallel-jtag-cable.png" caption="Xilinx Parallel Cable" %}

However, I find the method quite inconveinent as I do not always have access to a parallel port on every laptop or desktop. Besides, I could not afford any genuine Xilinx Platform USB Cable which costs around 250$ while the ripoff costs around 30$. So, I stopped my research on FPGA and moved on to other areas.

After 3 years, I decided to take on my research on FPGA again so I am using back my Xilinx Spartan-6 dev board. During my search for a cheap alternative method using USB to program the FPGA, I came across this [blog post](https://debugmo.de/2012/02/xvcd-the-xilinx-virtual-cable-daemon/) mentioning `xvcd` (Xilinx Virtual Cable Daemon) plugin in Xilinx iMPACT. The plugin enables iMPACT to upload bitstream files over TCP/IP rather than any Xilinx supported cables. Therefore, `tmbnic` from [debugmo.de](https://debugmo.de/) implemented [a client](https://github.com/tmbinc/xvcd/tree/ftdi) for FTDI cables that support JTAG interface to program the FPGA via USB.

I purchased [a FT2232H board](https://item.taobao.com/item.htm?id=571105722642) from Taobao.com which costs around 11$.

{% include image name="ft2232h.png" caption="FT2232H board" %}

Following the JTAG cable pinout provided by [nxlab](http://www.nxlab.fer.hr/fpgarduino/linux_bitstreams.html) and the FT2232H board [schematics](https://github.com/arm8686/FT2232HL-BOARD), hook up `TCLK`, `TDO`, `TMS` and `TDI` pins with jumper cables. Then, hook up `VCC` to a 3.3v supply pin and `GND` to a ground pin on the FT2232H board.

{% include image name="jtag-pinout.png" caption="JTAG Pinout" %}

{% include image name="setup.png" caption="Setup" %}

However, `tmbinc` client may not support FT2232H as it mentions FT2232C only (not yet tested). So, I use [playtag](https://github.com/pmaupin/playtag/) which I found out from a [blog post](https://numato.com/kb/programming-saturn-spartan-6-fpga-module-using-impact-via-xilinx-virtual-cable) on Numato Lab site. `playtag` is written in Python and it requires Python version 2 and FTDI D2XX driver installed to run.

After cloning playtag respository, run `discovery.py` in `tools/jtag/` with the `ftdi` option to verify that your computer recognizes the FT2232H board.

{% highlight console %}
~/playtag/tools/jtag/$ python discover.py ftdi
2 devices found:

[0]
        Flags = '2'
        Type = '6'
        ID = '67330064'
        LocID = '401'
        SerialNumber = 'FT3CEFS9A'
        Description = 'USB <-> Serial Converter A'
        ftHandle = 'None'

[1]
        Flags = '2'
        Type = '6'
        ID = '67330064'
        LocID = '402'
        SerialNumber = 'FT3CEFS9B'
        Description = 'USB <-> Serial Converter B'
        ftHandle = 'None'

You may specify device by index number, serial number or 
description.
{% endhighlight %}

Since FT2232H consists of 2 channel A and B, make sure you select the channel to which the JTAG interface is hooked. Run `xilinx_xvc.py` in the same directory with the `ftdi` option and the channel number. If no errors occur, the program will wait for the connection.

{% highlight console %}
~/playtag/tools/jtag/$ python xilinx_xvc.py ftdi 1
Waiting for xvc connection on 0.0.0.0:2542  (Ctrl-C to exit)
{% endhighlight %}

Now we are ready to program the FPGA. Go to `Output > Cable Setup...` on Xilinx iMPACT. Then, tick the `Open Cable Plug-in` checkbox at the bottom and type in `xilinx_xvc host=localhost:2542 disableversioncheck=true` in the text field.

{% include image name="cable-setup.png" %}
{% include image name="cable-plugin.png" %}

After pressing 'OK', the client will says `xvcd` has connected. Viola! You can start programming the FPGA as usual using Xilinx iMPACT. 

{% highlight console %}
~/playtag/tools/jtag/$ python xilinx_xvc.py ftdi 1
Waiting for xvc connection on 0.0.0.0:2542  (Ctrl-C to exit)
Conncted to 127.0.0.1:58061 -- now serving xvc
{% endhighlight %}

----
References
- <https://debugmo.de/2012/02/xvcd-the-xilinx-virtual-cable-daemon/>
- <http://www.nxlab.fer.hr/fpgarduino/linux_bitstreams.html>
- <https://numato.com/kb/programming-saturn-spartan-6-fpga-module-using-impact-via-xilinx-virtual-cable/>
- <https://github.com/arm8686/FT2232HL-Board/>



