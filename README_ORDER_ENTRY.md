# Manual Order Entry -- Liquibook example program

This program accepts requests typed from the console or read from a script file and applies them to
one or more order books.  It displays the results of the callbacks generated by Liquibook in
response to those requests.

## Request syntax

An empty line or a line beginning with a hash (#) is considered a comment and will be ignored.
Other lines read are in the format:

   REQUEST [parameters]* [;]

That is: a one word request followed by parameters as necessary and terminated in a semicolon.  

The semicolon may be optional for some commands, but it can always be included if desired.
The stand-alone word "END" may be substituted for a semicolon.

If a request accepts optional parameters, or if not all required parameters are provided, the 
console user will be prompted for additional parameters until a semicolon is entered.
This prompting will happen even if the input is being read from a file, so the requests coming
in from a script file should be complete and should be terminated with a semicolon.
   
####For example:

   > BUY 100 GOOG 850 AON;
       ADDING order:  [#2 BUY 100 GOOG $850 AON Last Event:{Submitted BUY 100 GOOG @850}]
   > BUY              (incomplete request)
   Quantity: 100      (prompt for parameters)
   Symbol: GOOG
   Limit Price or MKT: 800
   AON, or IOC, or STOP, or END: ;
       ADDING order:  [#3 BUY 100 GOOG $800 Last Event:{Submitted BUY 100 GOOG @800}]  
       
## Requests

Most requests can be spelled out or abbreviated to their first character.  
Parameters accepted by each request will be explained later in this document.
Requests are not case sensitive.  Even though they are shown here in all CAPS
a BUY request may be entered as BUY, buy, b, or even Buy.

* BUY or B      : Enter a new order to buy a security.
* SELL or S     : Enter a new order to sell a security.
* CANCEL or C   : Request that an existing order be canceled.
* MODIFY or M   : Request that an existing order be modified.
* DISPLAY or D  : Display information about existing orders
* FILE or F     : Open or close a script file.
* ?             : Display help for requests
* QUIT          : Exit the program.
   
## Symbols

Liquibook itself imposes no limitations on symbol.  It accepts an arbitrary string of characters 
(actually of 8 bit octets so UTF-8 or other encodings are supported with no special effort.)

The manual entry program has a few restrictions on the symbols to ease parsing: 
    * The symbol may not include spaces.
    * The symbol may not begin with a plus sign (+) or exclamation mark(!)
    * The symbol should not be ALL.
    * The symbol should not be a single asterisk (*)

It is important to stress that these restrictions apply only to the example program, not to Liquibook itself.

Disallowing spaces obviously makes parsing the incoming request easier.  
Using "ALL" and "*" would confict with the DISPLAY command parsing.
  
The restriction against leading plus signs and exclamation marks requires more explanation.

Liquibook uses a separate order book for each security.  When a new symbol is encountered (representing a new security)
the manual entry program will create a new order book for the symbol.  It needs to know whether to create
a simple order book, or a depth book.  

* If the new symbol is prefixed by a plus sign(+), then a simple
order book is created. 
* If it is prefixed by an exclamation mark (!) then a depth book is created.  
* If neither of these prefixes is present then manual entry will prompt the console user to determine what
type of book to create.

For example:

    > buy 100 ibm 50;
    New Symbol IBM.
    Add [S]imple book, or [D]epth book, or 'N' to cancel request.
    [SDN}: d
    Create new depth order book for IBM
    ADDING order:  [#1 BUY 100 IBM $50 Last Event:{Submitted BUY 100 IBM @50}]
            Accepted: [#1 BUY 100 IBM $50 Open: 100 Last Event:{Accepted }]
            Book Change:  IBM
            Depth Change:  IBM Changed Change Id: 1 Published: 0
            BIDS:
            Price 50 Count: 1 Quantity: 100 Change id#: 1
            BBO Change:  IBM Changed Change Id: 1 Published: 0
    > buy 200 t 78;
    New Symbol T.
    Add [S]imple book, or [D]epth book, or 'N' to cancel request.
    [SDN}: s
    Create new order book for T
    ADDING order:  [#2 BUY 200 T $78 Last Event:{Submitted BUY 200 T @78}]
            Accepted: [#2 BUY 200 T $78 Open: 200 Last Event:{Accepted }]
            Book Change:  T
    > buy 50 r 99;
    New Symbol R.
    Add [S]imple book, or [D]epth book, or 'N' to cancel request.
    [SDN}: n
    Request ignored
    Cannot process command BUY 50 R 99 ;

Since console  prompting does not play well with scripting, it is important that symbols used in scripts
have the appropriate prefix the first time they appear.

It is acceptable to have a + or ! prefix on a symbol for which a book already exists.  The prefix will be ignored
and the existing book will be used.

## Prices

Liquibook uses integers for prices.  It does care what currency unit is actually being used as long as all orders for a
particular security use the same units.   

Because Liquibook uses integers the prices must be expressed in terms of the "atomic currency unit"  For example
if prices are in US Dollars, then the atomic currency unit is often a penny, so fifteen dollars would be expressed to
Liquibook as 1500.

## Order Identity

Liquibook uses the memory address of the applications order object as an identifier for the order. [This may be changed in the future because it
imposes some odd restrictions on object lifetime.]  The manual entry program needs a way for the console user (or script)
to identify orders, and the memory address is obviously not available for this purpose.

Thus the order entry program assigns an order id to each order.  Whenever an order is displayed the order ID appears after a hash sign(#).
So for example if a new order is added, and the response is:

      ADDING order:  [#1 BUY 100 IBM $50 Last Event:{Submitted BUY 100 IBM @50}]

This order may be referred to in following commands by using the order id 1. When an order is read from the console or from a script,
an optional # can prefix the order id.  Thus:  "Cancel #1" and "CANCEL 1" refer to the same order.

Order ID's may also be entered with a leading minus sign.  These order IDs are relative to the NEXT order id that will be used.  
Thus "Cancel -1" would cancel the most recent order entered.  "Cancel -2" would cancel the order before that, and so on.

## Details of specific Requests
   
### BUY or B 
Enter a new order to buy a security.

Syntax:  BUY quantity symbol price [AON | IOC | STOP price]* ;

Parameters:
* *quantity* is number of shares (or other tradable units) to buy.
* *symbol* is an arbitrary character string.
  * See the *Symbol" section above for information about this string.
* *price* is the price the trader is willing to pay for each tradable unit of the security.
  * See the *Price* section above for details about expressing prices.
* AON  This optional parameter sets the All-Or-None condition for this order.
* IOC  This optional parameter sets the Immediate-Or-Cancel condition for this order.
* STOP price  This optional parameter sets a STOP LOSS price for this order using the same currency units as the price parameter.

The traling semicolon (or the word END) is required for a BUY request.

Zero or more trades may be triggered by entering a BUY order.

### SELL or S
Enter a new order to sell a security.

Syntax:  SELL quantity symbol price [AON | IOC | STOP price]* ;

Parameters:
* *quantity* is number of shares (or other tradable units) to be sold.
* *symbol* is an arbitrary character string.
  * See the *Symbol" section above for information about this string.
* *price* is the price the trader is willing to acce[t for each tradable unit of the security.  
  * See the *Price* section above for details about expressing prices.
* AON  This optional parameter sets the All-Or-None condition for this order.
* IOC  This optional parameter sets the Immediate-Or-Cancel condition for this order.
* STOP price  This optional parameter sets a STOP LOSS price for this order using the same currency units as the price parameter.

The traling semicolon (or the word END) is required for a SELL request.

Zero or more trades may be triggered by entering a SELL order.

### CANCEL or C
Request that an existing order be canceled.

Parameter:
* orderid, or #orderid, or -relative_order_id.
  * See the section on Order Identity above for details. 
  
No trades will be triggered by entering a CANCEL request.

The traling semicolon (or the word END) is optional for a CANCEL request. The request is considered complete after one required parameter.

The request may be rejected if the order has already been filled.

### MODIFY or M

Request that an existing order be modified.

Parameters:
* orderid, or #orderid, or -relative_order_id.
  * See the section on Order Identity above for details. 
* PRICE new_price
  * See the *Price* section above for details about expressing prices.
  * Prices may be increased or decreased.
  * PRICE is optional.  If it is not specified the existing price will be unchanged.
* QUANTITY new_initial_quantity
  * Not this is NOT the new quantity to be open (that's a moving target).  Nor is it a delta to quantity (although that's how Liquibook expresses it internally.)
  * QUANTITY is optional.  If it is not specified the existing quantity will be unchanged.

The traling semicolon (or the word END) is required for a MODIFY request.

Zero or more trades may be triggered by entering a SELL order.

The request may be rejected if the order has already been filled, or if no changes are requested.

### DISPLAY or D
Display information about existing orders

Parameters:
* * this optional first parameter requests a more verbose display.
* order_id or symbol or ALL.
  * One of these must appear.
  * See the *Order Identity" section above for a description of the ways in which order ID cam be expressed.
  * The literal word ALL will display information about all known symbols.

### FILE or F
Open or close a script file.

Parameter:
* If this command appears in a script file, no parameter is allowed. This is considered to be a request to close the script file.
* filename    If this command is entered from the console, the filename is the name of a script file from which future commands should com.

Commands will be read from the script file until:
* The "FILE" command appears in the script file,
* End-of-file on the script file.
* The "QUIT" command appears in the script file.

If the script file was originally opened by a FILE command entered from the console, and the script exits because of-
a FILE command or end-of-file, then control returs to the console.
If the script file was opened from the command line (see below), or the QUIT command appears in the script file, then the program exits.

### HELP or ?
Display help for requests

Displays a brief form of the information in this document.

### QUIT
Exit the program.

## The Command Line

The mt_order_entry command accepts two command line options.  Both are optional.

Parameter:
    script_file_name  The name of a script file or '-' if commands are to be read from the console.
    log_file_name  The name of a file to which output should be written.  Prompts (if any) will still be written to the console.)
    