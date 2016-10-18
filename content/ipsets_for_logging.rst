IPsets to log connections
#########################

:date: 2016-10-17
:tags: iptables
:category: sysadmin
:slug: ipset_conn_log
:author: Ryan Carter


Recently we had a server that needed to be decommissioned and it was one of those old servers everyone has (and always trying to get rid of) that runs too many important things.
We knew what processes were running sure, but wanted to be sure nothing was still connecting to it over a long period of time (say, a week). Trusty netfilter to the rescue.

ipsets have many different set types, in this case we used a `hash:ip,port` type.


.. code-block:: sh

   # Add our ipset with no expiry and a reasonably high maxelem (this is the default).
   $ ipset create server_connectors hash:ip,port family inet maxelem 65536 timeout 0
   
   # Add iptables rule to match new tcp connections on one interface and always try to add.
   $ iptables -I INPUT 2 -i $external_int -p tcp -m conntrack --ctstate NEW -j SET --add-set server_connectors src --exist

All good right?

.. code-block:: sh

   $ ipset save server_connectors | grep -v create | wc -l
   0

Uh... What?

Lets look at the man page for the SET target (iptables-extensions in debian)

.. 

   SET
       This module adds and/or deletes entries from IP sets which can be defined by ipset(8).

       --add-set setname flag[,flag...]
              add the address(es)/port(s) of the packet to the set

And in the ipset man page?

..

   TYPENAME := method:datatype[,datatype[,datatype]]

For whatever reason I did not associate "flag" with "datatype". /s

So now our rule looks like this:

.. code-block:: sh

   $ iptables -I INPUT 2 -i $external_int -p tcp -m conntrack --ctstate NEW -j SET --add-set server_connectors src,dst --exist

And so we have our list populating

.. code-block:: sh

   $ ipset save server_connectors | grep -v create | wc -l
   23
