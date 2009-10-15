---
layout: default
title: Working With the Clipboard
---

<h2>Working With the Clipboard</h2>

Two of the standard FOX widgets, <classname>FXText</classname> and
<classname>FXTextField</classname>, provide clipboard support out of the
box. For example, you can select some text in an
<classname>FXTextField</classname> and then press Ctrl+C to copy that text
to the system clipboard. You can also press Ctrl+X to "cut" the selected
text to the clipboard, or Ctrl+V to paste text from the clipboard into an
<classname>FXText</classname> or <classname>FXTextField</classname> widget.
The purpose of this tutorial is to demonstrate how to interact with the
clipboard programmatically, so that you can integrate additional clipboard
support into your FXRuby applications.

Basic Application
-----------------

In order to illustrate how to integrate cut and paste operations
into your application, we'll start from a simple FXRuby application that
doesn't yet provide any clipboard support. This application simply
presents a list of customers (from some external source).

{% highlight ruby %}
require 'fox16'
require 'customer'

include Fox

class ClipMainWindow &lt; FXMainWindow
  def initialize(anApp)
    # Initialize base class first
    super(anApp, "Clipboard Example", :opts => DECOR_ALL, :width => 400, :height => 300)

    # Place the list in a sunken frame
    sunkenFrame = FXVerticalFrame.new(self,
			LAYOUT_FILL_X|LAYOUT_FILL_Y|FRAME_SUNKEN|FRAME_THICK, :padding => 0)
    
    # Customer list
    customerList = FXList.new(sunkenFrame, :opts => LIST_BROWSESELECT|LAYOUT_FILL_X|LAYOUT_FILL_Y)
    $customers.each do |customer|
      customerList.appendItem(customer.name, nil, customer)
    end
  end
  
  def create
    super
    show(PLACEMENT_SCREEN)
  end
end

if __FILE__ == $0
  FXApp.new("ClipboardExample", "FXRuby") do |theApp|
    ClipMainWindow.new(theApp)
    theApp.create
    theApp.run
  end
end
{% endhighlight %}

We're assuming that the "customer" module defines a
<classname>Customer</classname> class and a global array
<varname>$customers</varname> that contains the list of customers. For a
real world application, you might access this information from a database
or some other source, but for this example we'll just use a hard-coded
array:

{% highlight ruby %}
# customer.rb

Customer = Struct.new("Customer", :name, :address, :zip)

$customers = []
$customers << Customer.new("Reed Richards", "123 Maple, Central City, NY", 010111)
$customers << Customer.new("Sue Storm", "123 Maple, Anytown, NC", 12345)
$customers << Customer.new("Benjamin J. Grimm", "123 Maple, Anytown, NC", 12345)
$customers << Customer.new("Johnny Storm", "123 Maple, Anytown, NC", 12345)
{% endhighlight %}

The goals for the next few sections are to extend this application
so that users can select a customer from the list and copy that customer's
information to the clipboard, and subsequently paste that information into
another copy of the program (or some other clipboard-aware
application).

Acquiring the Clipboard
-----------------------

Let's begin by augmenting the GUI to include a row of buttons along
the bottom of the main window for copying and pasting:

{% highlight ruby %}
require 'customer'

include Fox

class ClipMainWindow < FXMainWindow
  def initialize(anApp)
    # Initialize base class first
    super(anApp, "Clipboard Example", :opts => DECOR_ALL, :width => 400, :height => 300)
<bold>
    # Horizontal frame contains buttons
    buttons = FXHorizontalFrame.new(self, LAYOUT_SIDE_BOTTOM|LAYOUT_FILL_X|PACK_UNIFORM_WIDTH)

    # Cut and paste buttons
    copyButton  = FXButton.new(buttons, "Copy")
    pasteButton = FXButton.new(buttons, "Paste")
</bold>
    # Place the list in a sunken frame
    sunkenFrame = FXVerticalFrame.new(self,
			LAYOUT_FILL_X|LAYOUT_FILL_Y|FRAME_SUNKEN|FRAME_THICK, :padding => 0)
    
    # Customer list
    customerList = FXList.new(sunkenFrame, :opts => LIST_BROWSESELECT|LAYOUT_FILL_X|LAYOUT_FILL_Y)
    $customers.each do |customer|
      customerList.appendItem(customer.name, nil, customer)
    end
  end
  
  def create
    super
    show(PLACEMENT_SCREEN)
  end
end

if __FILE__ == $0
  FXApp.new("ClipboardExample", "FXRuby") do |theApp|
    ClipMainWindow.new(theApp)
    theApp.create
    theApp.run
  end
end
{% endhighlight %}

Note that the lines which appear in bold face are those which have
been added (or changed) since the previous source code listing.

The clipboard is a kind of shared resource in the operating system.
Copying (or cutting) data to the clipboard begins with some window in your
application requesting "ownership" of the clipboard by calling the
<methodname>acquireClipboard()</methodname> instance method. Let's add a
handler for the "Copy" button press which does just that:

{% highlight ruby %}
# User clicks Copy
copyButton.connect(SEL_COMMAND) do
  customer = customerList.getItemData(customerList.currentItem)
  types = [ FXWindow.stringType ]
  if acquireClipboard(types)
    @clippedCustomer = customer
  end
end
{% endhighlight %}

<para>The <methodname>acquireClipboard()</methodname> method takes as its
input an array of drag types. A <emphasis>drag type</emphasis> is just a
unique value, assigned by the window system, that identifies a particular
kind of data. In this case, we're using one of FOX's pre-registered drag
types (<constant>stringType</constant>) to indicate that we have some
string data to place on the clipboard. Later, we'll see how to register
customized, application-specific drag types as well.</para>

<para>The <methodname>acquireClipboard()</methodname> method returns
<constant>true</constant> on success; since we called
<methodname>acquireClipboard()</methodname> on the main window, this means
that the main window is now the clipboard owner. At this time, we want to
save a reference to the currently selected customer in the
<varname>@clippedCustomer</varname> instance variable so that if its value
is requested later, we'll be able to return the
<emphasis>correct</emphasis> customer's information.</para>

Sending Data to the Clipboard
-----------------------------

Whenever some other window requests the clipboard's contents (e.g.
as a result of a "paste" operation) FOX will send a
<constant>SEL_CLIPBOARD_REQUEST</constant> message to the current
clipboard owner. Remember, the clipboard owner is the window that called
<methodname>acquireClipboard()</methodname>. For our example, the main
window is acting as the clipboard owner and so it needs to handle the
<constant>SEL_CLIPBOARD_REQUEST</constant> message:

{% highlight ruby %}
# Handle clipboard request
self.connect(SEL_CLIPBOARD_REQUEST) do
  setDNDData(FROM_CLIPBOARD, FXWindow.stringType, Fox.fxencodeStringData(@clippedCustomer.to_s))
end
{% endhighlight %}

The <methodname>setDNDData()</methodname> method takes three
arguments. The first argument tells FOX which kind of data transfer we're
trying to accomplish; as it turns out, this method can be used for
drag-and-drop (<constant>FROM_DRAGNDROP</constant>) and X11 selection
(<constant>FROM_SELECTION</constant>) data transfer as well. The second
argument to <methodname>setDNDData()</methodname> is the drag type for the
data and the last argument is the data itself, a binary string.

If you're wondering why we need to call the
<methodname>fxencodeStringData()</methodname> module method to preprocess
the string returned by the call to <methodname>Customer#to_s</methodname>,
that's a reasonable thing to wonder about. In order for FOX to play nice
with other clipboard-aware applications, it must be able to store string
data on the clipboard in the format expected by those applications.
Unfortunately, that expected format is platform-dependent and does not
always correspond directly to the format that Ruby uses internally to
store its string data. The <methodname>fxencodeStringData()</methodname>
method (and the corresponding
<methodname>fxdecodeStringData()</methodname> method) provide you with a
platform-independent way of sending (or receiving) string data with the
<constant>stringType</constant> drag type.

If you run the program as it currently stands, you should now be
able to select a customer from the list, click the "Copy" button and then
paste the selected customer data (as a string) into some other
application. For example, if you're trying this tutorial on a Windows
machine, try pasting into a copy of Notepad or Microsoft Word. The pasted
text should look something like:

{% highlight ruby %}
#<struct Struct::Customer name="Joe Smith", address="123 Maple, Anytown, NC", zip=12345>
{% endhighlight %}

Pasting Data from the Clipboard
-------------------------------

We've seen one side of the equation, copying string data to the
clipboard. But before we can "round-trip" that customer data and paste it
back into another copy of our customer list application, we're clearly
going to need to transfer the data in some more useful format. That is to
say, if we were to receive the customer data in the format that it's
currently stored on the clipboard:

{% highlight ruby %}
#<struct Struct::Customer name="Joe Smith", address="123 Maple, Anytown, NC", zip=12345>
{% endhighlight %}

We'd have to parse that string and try to extract the relevant data
from it. We can do better than that. The approach we'll use instead is to
serialize and deserialize the objects using YAML. First, make sure that
the YAML module is loaded by adding this line:

{% highlight ruby %}
require 'yaml'
{% endhighlight %}

somewhere near the top of the program. Next, register a custom drag
type for <classname>Customer</classname> objects. We can do that by adding
one line to our main window's <methodname>create</methodname> instance
method:

{% highlight ruby %}
def create
  super
  <bold>@customerDragType = getApp().registerDragType("application/x-customer")</bold>
  show(PLACEMENT_SCREEN)
end
{% endhighlight %}

Note that by convention, the name of the drag type is the MIME type
for the data, but any unique string will do. In our case, we'll use the
string "application/x-customer" to identify the drag type for our
YAML-serialized <classname>Customer</classname> objects.

With that in place, we can now go back and slightly change some of
our previous code. When we acquire the clipboard, we'd now like to be able
to offer the selected customer's information either as plain text (i.e.
the previous format) <emphasis>or</emphasis> as a YAML document, so we'll
include <emphasis>both</emphasis> of these types in the array of drag
types passed to <methodname>acquireClipboard()</methodname>:

{% highlight ruby %}
# User clicks Copy
copyButton.connect(SEL_COMMAND) do
  customer = customerList.getItemData(customerList.currentItem)
  <bold>types = [ FXWindow.stringType, @customerDragType ]</bold>
  if acquireClipboard(types)
    @clippedCustomer = customer
  end
end
{% endhighlight %}

Similarly, when we're handling the
<constant>SEL_CLIPBOARD_REQUEST</constant> message, we now need to pay
attention to which drag type (i.e. which data format) the requestor
specified. We can do that by inspecting the
<methodname>target</methodname> attribute of the
<classname>FXEvent</classname> instance passed along with the
<constant>SEL_CLIPBOARD_REQUEST</constant> message:

{% highlight ruby %}
# Handle clipboard request
self.connect(SEL_CLIPBOARD_REQUEST) do |sender, sel, event|
  case event.target
    when FXWindow.stringType
      setDNDData(FROM_CLIPBOARD, FXWindow.stringType, Fox.fxencodeStringData(@clippedCustomer.to_s))
    when @customerDragType
      setDNDData(FROM_CLIPBOARD, @customerDragType, @clippedCustomer.to_yaml)
    else
      # Ignore requests for unrecognized drag types
  end
end
{% endhighlight %}

With these changes in place, we can now add a handler for the
"Paste" button which requests the clipboard data in YAML format,
deserializes it, and then adds an item to the customer list:

{% highlight ruby %}
# User clicks Paste
pasteButton.connect(SEL_COMMAND) do
  data = getDNDData(FROM_CLIPBOARD, @customerDragType)
  if data
    customer = YAML.load(data)
    customerList.appendItem(customer.name, nil, customer)
  end
end
{% endhighlight %}

The <methodname>getDNDData()</methodname> method used here is the
inverse of the <methodname>setDNDData()</methodname> method we used
earlier to push data to some other application requesting our clipboard
data. As with <methodname>setDNDData()</methodname>, the arguments to
<methodname>getDNDData()</methodname> indicate the kind of data transfer
we're performing (e.g. <constant>FROM_CLIPBOARD</constant>) and the drag
type for the data we're requesting. If some failure occurs (usually,
because the clipboard owner can't provide its data in the requested
format) <methodname>getDNDData()</methodname> will simply return
<constant>nil</constant>.
