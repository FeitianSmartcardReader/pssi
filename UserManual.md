# Introduction #

In this wiki page, you will find some indications about how to use PSSI and how to write your own plugin.


# How to use the tool #

In order to run PSSI, make sure `pyscard` and the `PSSI` packages are installed. You can then run `pssi.py`, without forgetting to specify a plugin, and read your smartcards!

<pre>
Usage: pssi  [options] plugin_path<br>
Example: ./pssi.py plugins/sim<br>
-a        apdu mode, show the APDUs<br>
-b        choose bruteforce mode<br>
-c        specify the class byte for the bruteforce mode, in hexadecimal<br>
-d        choose dump mode (default, specify a plugin)<br>
-h        show this help<br>
-l        choose loop mode (specify a plugin)<br>
-r        enable recursive mode in the bruteforce mode<br>
-v        verbose mode, show the raw data along with the interpretation<br>
</pre>

Here are some examples:
  * $ ./pssi.py plugins/emv
  * $ ./pssi.py plugins/navigo
  * $ ./pssi.py plugins/sim

# How to write a plugin #

## Setting up the environment ##

  * Make a clean directory
  * Write your `plugin.py`, with the following functions:
    * _getClassByte_: This functions returns the class byte of your smart card, which is the first byte sent in every APDU.
    * _getRootStructure_: This functions returns the root structure of your smartcard.
    * _getInterpretersTable_: This functions returns the interpreters table, which makes a correspondance between a data type and an interpreting function. An interpreting function is used to convert raw data (bytes or bits) into meaningful and human readable information.


## Adding some strucures ##

A structure describes a file in the smartcard file hierarchy, by listing its elements. It consists of an array of tuples, whose number depends on wether you are describing a DF (Dedicated File), such as the MF (Master File), or an EF (Elementary file).

For a DF, you use 4-tuples, each one containing:
  * The name of the element, which will be part of the final output
  * Its type, which is a member of the `FieldType` class
  * Its relative address, as a 2-byte-array
  * The name of the Python structure detailing the contents of the element. If the TLV (Tag-Length-Value) scheme must be used, this value is -1

There is a special case, present in the EMV card for instance, where an EF can contain a series of structures, whose number and adresses are not known in advance. In this case, the type is `FieldType.DFList`, and, if the name of the Python structure is -1, a fifth member of the tuple can be defined, which specifies a default structure if no tag is found, or if the tag is not a key of the interpreting table.

Here is an example:
```
structSIM = [
    ("ICC identification", FieldType.TransparentEF, [0x2f, 0xe2], structICC),
    ("DF GSM", FieldType.DF, [0x7f, 0x20], structGSM),
    ("DF Télécom", FieldType.DF, [0x7f, 0x10], structDFTel),
]
```

For an EF, you use 5-tuples, each one containing:
  * The name of the element, which will be part of the final output
  * Its type, which is a member of the `FieldType` class
  * Its size, usually in bytes. If the element doesn't have a fixed size, this number can be negative; it then specifies the amount of data following the current element, which has to be fixed
  * A comment, which will be part of the final output, in order to make it for user friendly. It can be empty
  * The nature of the element, which allows PSSI to know how to interpret the element, as described in the next section

Here is an example:
```
structNumber = [
    ("Alpha identifier", FieldType.Final, -14, "Name of the contact", FinalType.Contact),
    ("Length of relevant information", FieldType.Final, 1, "In bytes", FinalType.Integer),
    ("TON and NPI", FieldType.Final, 1, "", FinalType.TonNpi),
    ("Dialling number", FieldType.Final, 10, "Telephone number of the contact", FinalType.NumRevHexString),
    ("Capability/Configuration identifier", FieldType.Final, 1, "", FinalType.Integer),
    ("Extension1 record identifier", FieldType.Final, 1, "", FinalType.Integer),
]
```

## Adding some interpreters ##

As explained above, the interpreting table maps interpreting functions to types of elements.
Sometimes, for EMV cards for instance, smartcards have their content encoded according to the TLV (Tag-Length-Value) scheme. In this case, it is possible to use the interpreting table to map interpreting functions to TLV tags.

The interpreting table can also be used to define some hooks, by associating functions to the following strings (see the Navigo plugin for an example):
  * "ANY" --> This hook is always executed after the interpretation of an element. It can be used to maintain global variables for instance. It takes four arguments:
    * The name of the element
    * Its interpretation
    * Its type, which is the key in the interpreting table
    * Its value, which is the raw information, before the interpretation
  * "ATR" --> This is hook is used to parse the `pyscard` ATR structure, and should return a string describing it
  * "FORMAT" --> This hook is executed before the interpretation of any element, in order to pre-format it. It takes an array of byte as an argument, and should return an object which can be parsed by the interpreting functions. FOr instance, it might be easier to deal with bit strings than with byte arrays, such as in the Navigo case.