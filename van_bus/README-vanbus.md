# VAN bus

The VAN bus is a communication protocol developed by the PSA Group (Peugeot, Citroen) and Renault. It was used by Peugeot and Citroen from around 1998 to 2005 in cars such as the Peugeot 206 and Citroen Xsara for communication between comfort related systems (Car radio, display, gauges, A/C, etc.). Because of EU regulation CAN bus had to be used for engine management and diagnostics, so the communication network was split between these two protocols. Such system was later abandoned in favor of a full CAN network.

## Reading the bus

The electrical signals on the VAN bus are the same as CAN, so a widely available CAN transciever can be used to sniff communication. However, due to way different messaging protocols, a VAN driver chip is required for sending data to the bus. These chips are the TSS463 and TSS461 made by Atmel, and they are difficult to find sometimes. Worst case scenario, they can be recovered from broken displays or headunits.

The bus itself is a bit complex, so here are a couple documents for background reading:

- [Wiki page on VAN](https://en.wikipedia.org/wiki/Vehicle_Area_Network)
- [VAN line protocol - Graham Auld - November 27, 2011](http://graham.auld.me.uk/projects/vanbus/lineprotocol.html)
- [Collection of data sheets - Graham Auld](http://graham.auld.me.uk/projects/vanbus/datasheets/)
- [Lecture on Enhanced Manchester Coding (in French) - Alain Chautar - June 30, 2003](http://www.educauto.org/files/file_fields/2013/11/18/mux1.pdf)
- [Lecture on VAN bus access: collisions and arbitration (in French) - Alain Chautar - October 10, 2003](http://www.educauto.org/sites/www.educauto.org/files/file_fields/2013/11/18/mux2.pdf)
- [Lecture on frame format (in French) - Alain Chautar - January 22, 2004](http://www.educauto.org/files/file_fields/2013/11/18/mux3.pdf)
- [Industrial networks CAN / VAN - Master Course - Marie-Agnès Peraldi-Frati - January 2008](http://www.i3s.unice.fr/~map/Cours/MASTER_STIC_SE/COURS32007.pdf)
- [De CAN à CANopen en passant par VAN (in French) - Jean Mercklé - March 2006](http://ebajic.free.fr/Ecole%20Printemps%20Reseau%20Mars%202006/Supports/J%20MERCKLE%20CANopen.pdf)
- [Les réseaux VAN - CAN (in French) - Guerrin Guillaume, Guers Jérôme, Guinchard Sébastien - February 2005](http://igm.univ-mlv.fr/~duris/NTREZO/20042005/Guerrin-Guers-Guinchard-VAN-CAN-rapport.pdf)
- [Le bus VAN, vehicle area network: Fondements du protocole (French) Paperback – June 4, 1997](https://www.amazon.com/bus-VAN-vehicle-area-network/dp/2100031600)
- [Vehicle Area Network (VAN bus) Analyzer for Saleae USB logic analyzer - Peter Pinter](https://github.com/morcibacsi/VanAnalyzer/)
- [Atmel TSS463C VAN Data Link Controller with Serial Interface](http://ww1.microchip.com/downloads/en/DeviceDoc/doc7601.pdf)
- [Multiplexed BSI Operating Principle for the Xsara Picasso And Xsara - The VAN protocol](http://milajda22.sweb.cz/Manual_k_ridici_jednotce.pdf#page=17)

And some helpful github repositories

- <https://github.com/0xCAFEDECAF/VanBus>
- <https://github.com/morcibacsi/arduino_tss463_van>
- <https://github.com/morcibacsi/PSAVanCanBridge>
- <https://github.com/0xCAFEDECAF/VanLiveConnect> - This repository has the most messages decoded, but they are buried in code.

## Message formats

The next big problem is reverse engineering the data inside the messages. Thankfully, [Peter Pinter](https://github.com/morcibacsi) has done a great job of reverse engineering most of the packets and created a [website](http://pinterpeti.hu/psavanbus/PSA-VAN.html) listing all currently known messages.

With help from [0xCAFEDECAF](https://github.com/0xCAFEDECAF), who decoded even more messages, I was able to read all the information I needed to make this work.

All used packet formats are located in the ESP source as c structs.

## Reading and parsing on the ESP

For reading, see the `van_receive_task` task function inside the ESP's firmware.  
For parsing, see `van_packet_parser.ino`.

All relevant code should be properly commented.
