## Optional Emonhub modifications

1. Modifying emonhub to post to emoncms node module for remote node decoding
2. Modifying emonhub to work with the emoncms packetgen module for sending out control packets over the rfm12/69 network.

### Using Emonhub with the optional emoncms node module

Emonhub has a powerfull node decoder built into it that allows for the decoding of node data with non-default packet structures. This decoder can be configured from emonhub.conf. There are ideas for how it could be possible to set the node decoders in the emonhub remotely but this is still a feature in early development. See emonhub issue 64:

[https://github.com/emonhub/emonhub/issues/64](https://github.com/emonhub/emonhub/issues/64)

In the meantime if you wish to carry out the node decoding within a remote emoncms installation a couple of small modifications to emonhub can be made to achieve this:

In emonhub.conf under listeners -> runtime_settings add the line

    defaultdatacode = b

In src/emonhub_dispatcher.py change [line 233](https://github.com/emonhub/emonhub/blob/development/src/emonhub_dispatcher.py#L233)

    post_url = self._settings['url']+'/input/bulk'+'.json?apikey='

to

    post_url = self._settings['url']+'/node/multiple'+'.json?apikey='
    
and change [line 247](https://github.com/emonhub/emonhub/blob/development/src/emonhub_dispatcher.py#L247)

    if reply == 'ok':
    
to

    if reply == 'true':

Restart emonhub to finish:

    sudo service emonhub restart
    
Check that there are no errors in the log:

    tail -f /var/log/emonhub.conf
    
If the emoncms node module is not present in your emoncms installation (if your using the bufferedwrite branch) then the node module can be installed from git by running in your emoncms/Modules folder:

    git clone https://github.com/emoncms/node.git
    
Complete the node module installation by running db update from within the admin panel of your emoncms account.

### Using Emonhub with the emoncms PacketGen module

The emoncms packetgen module can be used to construct control packets to be send out over the rfm12/69 network. The control packet is a register of properties that any node can pick and choose from. These properties could be room set point temperatures for radiator control nodes to make use of or lighting on/off etc. 

Development of control functionality within emoncms is at an early stage and we are currently discussing the best way to interface emonhub with emoncms for control, see: [https://github.com/emonhub/emonhub/issues/64](https://github.com/emonhub/emonhub/issues/64). The following is a quick modification that can be done to emonhub to get control working using packetgen until a more permanent solution is reached:

Start by installing packetgen by running the following in your emoncms/Modules folder:

    git clone https://github.com/emoncms/packetgen.git
    
Complete the packetgen module installation by running db update from within the admin panel of your emoncms account

We can modify emonhub to poll the packetgen module periodically and send the packetgen packet state over serial to the rfm12/69.

    cd /home/pi/emonhub/src
    
    nano emonhub_listener.py
    
Add just below import select [~line 16](https://github.com/emonhub/emonhub/blob/development/src/emonhub_listener.py#L16) the line:

    import urllib2
    
Add just below self._interval_timestamp = 0 [~line 48](https://github.com/emonhub/emonhub/blob/development/src/emonhub_listener.py#L48) the line:

    self._control_timestamp = 0
    
In class EmonHubJeeListener, method run, add just below: now = time.time() [~line 458](https://github.com/emonhub/emonhub/blob/development/src/emonhub_listener.py#L458)

    if now - self._control_timestamp > 5:
        self._control_timestamp = now
        packet = urllib2.urlopen("http://localhost/emoncms/packetgen/rfpacket.json?apikey=APIKEY").read()
        packet = packet[1:-1]
        self._log.debug(self.name + " broadcasting control packet " + packet + "s")
        self._ser.write(packet+"s")
        
Set your emoncms location (it can be locahost or a remote server) and apikey in the URL string.

Save and exit.
        
Restart emonhub

    sudo service emonhub restart