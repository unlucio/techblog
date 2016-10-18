---
layout: post
title:  Speeding up mobile apps publishing chores
author: Luciano Colosio
date:   2016-10-17
categories: javascript, automation, appstore
summary: automating appstore screenshots
---

> TLTR: When facing long and repetitive tasks, automation is one of our best assets and friends, and team communication is the key for a greater level of efficiency.

Publishing a mobile application, on the different stores out there, it can quickly turn into a chore.<br/>
A single set of screenshots will be fine at the begunning, but delivering quality content for all the different stores might require a little more effort and you might quickly end up having a person spending hours on this.<br/>
With 47 different supported languages here at Badoo, we turned to automation to help us speed up this process.

Our designer's tool of choice, [Sketch][1], happens to have a nice [plugin support][2] throught [Cocoalscript][3]: an Objective-C/[Cocoa][4] bridge to javascript.

So we quickly set off with a simple idea: read a bunch of text, find a bunch of images and apply them a number of times to the current sketch file.<br/>

### Sourcing the Copy
The 1st step was to find an efficient way to source the needed translations. Luckly for us our teams happen to already have in place a google spreadsheet to share translations between translators and designers:

![Translations Spreadsheet Example]({{page.imgdir}}/translationSheet.png)

There are times where for sake of semplicity, and if your data allow it (aka: is __not__ sensible data), simply reading data directly from a google doc ends up been your best choice, so: Let google docs be our backend!


As you can see our [cocoalscript][3] fetching function looks (mostly) javascript(-ish):
{% highlight javascript %}
function fetch(address) {
    var url = NSURL.URLWithString(address);

    var error = nil;
    var content = [NSString stringWithContentsOfURL:url encoding:NSUTF8StringEncoding error: error];

    if (error) {
        log('FETCH ERROR: ' + error);
        return nil;
    }

    return content;
}
{% endhighlight %}

(Seriusly: I love [cocoa][4]. The overwhelming elegance and semplicity of ``` [NSString stringWithContentsOfURL...]``` it always feels so good.)

Both javascript and Objc notation are allowed in cocoascript, with one catch: nested ```[]``` invocation will leave you baffled in front of an error.

So the familiar:

{% highlight obj-c %}
NSString * content = [NSString stringWithContentsOfURL:[NSURL URLWithString:address] encoding:NSUTF8StringEncoding error: &error];
{% endhighlight %}

Will crash saying it cannot find the requested element in the array.

The same statment can be also written with dot notation in the following form:

{% highlight javascript %}
var content = NSString.stringWithContentsOfURL_encoding_error(url, NSUTF8StringEncoding, error);
{% endhighlight %}

but it looks quite odd and ugly, and it doesn't always work, so the roule of thumb for our plugin was: use dot notation as much as possible for sake of portability (it's a JS developers team) unless you really need to do it in another way.

On the Google doc's side we chose to go with API V3 since, even been deprecated, it allowed us for quick data fetching without the need of implementing an utentication flow and 3 simple function let's us get all the data we need:

{% highlight javascript %}
function getSpreadsheetCellsAddress(spreadsheetId) {
    return 'https://spreadsheets.google.com/feeds/cells/' + spreadsheetId + '/od6/public/basic?alt=json';
}

function getSpreadsheetListsAddress(spreadsheetId) {
    return 'https://spreadsheets.google.com/feeds/list/' + spreadsheetId + '/od6/public/basic?alt=json';
}

function getWorkSheetsAddress(spreadsheetId) {
    return 'https://spreadsheets.google.com/feeds/worksheets/' + spreadsheetId + '/public/basic?alt=json';
}
{% endhighlight %}<br/>

### Creating a UI
Storyboard (interface builder, for those as old as I am) is clearly not an option, so: hello Xcode.

In short the esiest way to build UI parts was to code them in Xcode, with all the bells and wistles of a dedicated IDE

{% highlight obj-c %}
- (NSString *)promptWithMessage: (NSString *)msg defaultValue: (NSString *)defaultValue {
    NSAlert *alert = [[NSAlert alloc] init];
    [alert setMessageText: msg];
    [alert addButtonWithTitle:@"OK"];
    [alert addButtonWithTitle:@"CANCEL"];
    NSTextField *input = [[NSTextField alloc] initWithFrame:NSMakeRect(0, 0, 200, 24)];
    [input setPlaceholderString:defaultValue];
    [alert setAccessoryView:input];

    NSInteger button = [alert runModal];

    if (button == NSAlertFirstButtonReturn) {
        [input validateEditing];
        return [input stringValue];
    } else if (button == NSAlertSecondButtonReturn) {
        return nil;
    } else {
        return nil;
    }
}
{% endhighlight %}

and then extract and translate:

{% highlight javascript %}
function prompt(msg, defaultValue) {
    msg = msg || 'Please add a message';
    defaultValue = defaultValue || '';
    var alert = NSAlert.alloc().init();
    alert.setMessageText(msg);
    alert.addButtonWithTitle('OK');
    alert.addButtonWithTitle('CANCEL');
    var input = NSTextField.alloc().initWithFrame(NSMakeRect(0, 0, 400, 24));
    input.setPlaceholderString(defaultValue);
    alert.setAccessoryView(input);
    var pressedButton = alert.runModal();
    var result = nil;

    if (pressedButton === NSAlertFirstButtonReturn) {
        result = input.stringValue();
    }

    if (pressedButton === NSAlertSecondButtonReturn) {
        result = -1
    }

    return (result != '') ? result : nil;
}
{% endhighlight %}

And here's a new weird thing: we found out that due to string VS NSString type difference we cannot do a strict string comparison, so mostly forget about ```!==``` and ```===```
We soon hit a number of other small problems here and there but the most frustating was defintelly the impossibility of using delegates:
even trying with [MochaJSDelegate][5], something kept preventing the correct selector form been found. We never ofund a work around this.

### Localizing the art boards
Once we were able to communicate with our user and fetch waht we needed form google, it was finally time to start putting things together.
Artboards and layers are easily accessible via:

{% highlight javascript %}
var boards = context.document.currentPage().artboards();
var layer = boards[0].layers();
{% endhighlight %}

That's great, but as soon as we started using them we soon found out that javascript and Objc are so mixed that at any point in time we could end up dealing with either a javascript map/array or with an Objc NSDictinary/NSArray. This brought us to build a tiny "lodash" for sake of convenince and we'd love to share it's base foreach:

{% highlight javascript %}
_.forEach = function (subject, cb) {
    subject = subject || {};

    function each(limit, sub) {
        for (var i = 0; i < limit; i++) {
            if (subject.objectAtIndex) {
                cb(subject.objectAtIndex(i), i);
            } else {
                cb(subject[i], i);
            }
        }
    }

    function objectEach(sub) {
        for (var key in sub) {
            cb(sub[key], key);
        }
    }

    if (subject.count)  {
        each(subject.count(), subject);
        return;
    }

    if (subject.length) {
        each(subject.length, subject);
        return;
    }

    if (Object.keys(subject).length > 0) {
        objectEach(subject);
        return;
    }
}
{% endhighlight %}

With the power of ```forEach()```, ```map()```, ```reduce()``` and ```filter()``` applying the needed traslations (and custom pictures) was quite streight forward:
find the needed artboard, copy it, apply translations and be ready for exporting. Almost.
All our copied arboards were ending up in the same postion (one on top of each other) and the export result was the top most image, aka the last translated version.
Duplucating the artboard (appearenlty the same action obtained with ```command+d```) rather than copying it didn't help at all, so here's a littel function to allign all your copied artboard based on the same name:

{% highlight javascript %}
function alignBoardClonesOf(originName, spacing) {
    spacing = spacing || 150;
    var board = getBoardByName(originName);
    var clones = getBoardsByPrefix(originName);
    var frame = board.frame();
    var origin = [frame y];
    var step = [frame height];

    _.forEach(clones, function (clone) {
        if (clone.name() == originName) {
            return;
        }

        origin = origin + step + spacing;

        clone.frame().setY(origin);
    });
}
{% endhighlight %}<br/>

### Exporting the final pngs
Aligned our artboard we were now ready to export the final product:

{% highlight javascript %}
function saveArtboardToFile(artboard, basePath, options) {
    options = options || {};
    options.scale = options.scale || 1;
    options.format = options.format || 'png';
    var rect = artboard.absoluteRect().rect();
    var slice = [MSExportRequest requestWithRect:rect scale:options.scale];
    var file = getArtboardFilename(artboard.name());
    var path = basePath + '/' + file.langPath;
    var file = path + '/' + file.name + '.' + options.format;

    slice.setSaveForWeb(1);
    context.document.saveExportRequest_toFile(slice, file);
}
{% endhighlight %}<br/>

### Debugging and Documentation
The only debug mean is the built in ```log()``` function, that wraps ```NSLog()``` and ends up writing in the system logs: logs are then accessible from the Console.app filtering for "Sketch".
Documentation at the point we build the plugin was pretty minimal and it looked a lot as it was under rebuild (and thus kind of incomplete).
Our best friends in figuring out all the different avvailable objects and their APIs were [class-dump][6] and this very helpful little inspect function:

{% highlight javascript %}
function inspect(obj) {
  log('______________________________________');
  log('Inspecting object ' + obj );
  log('______________________________________');
  log('obj.properties: ' + [obj class].mocha().properties());
  log('obj.propertiesWithAncestors:'+ [obj class].mocha().propertiesWithAncestors());
  log('obj.classMethods:' + [obj class].mocha().classMethods());
  log('obj.classMethodsWithAncestors:' + [obj class].mocha().classMethodsWithAncestors());
  log('obj.instanceMethods:' + [obj class].mocha().instanceMethods());
  log('obj.instanceMethodsWithAncestors:' + [obj class].mocha().instanceMethodsWithAncestors());
  log('obj.protocols:' + [obj class].mocha().protocols());
  log('obj.protocolsWithAncestors:' + [obj class].mocha().protocolsWithAncestors());
  log('obj.treeAsDictionary():' + obj.treeAsDictionary());
}
{% endhighlight %}<br/>

### Conclusions and take aways
The whole adventure lasted on and off a couple of weeks and the heavy lifting was mainly in figuring out [Cocoalscript][3]'s peculiarities as well as finding the parts of the developer API we needed.
Some developer time and some good inter-team collabolariont allowed us to be now able to go from several days to a few seconds to obtain all the images we need for our appstore publications.

### And finally the result:
------- Lucio Make a GIF showing the plugin working -------

[1]: https://sketchapp.com/
[2]: http://developer.sketchapp.com/
[3]: https://github.com/ccgus/CocoaScript
[4]: https://en.wikipedia.org/wiki/Cocoa_(API)
[5]: https://github.com/matt-curtis/MochaJSDelegate
[6]: https://github.com/nygard/class-dump
