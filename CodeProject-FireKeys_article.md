[![Home](https://codeproject.freetls.fastly.net/App_Themes/CodeProject/Img/logo250x135.gif "CodeProject")](https://www.codeproject.com/)

Articles / [Desktop Programming / WPF

#csharp #windows #win32 #wpf

# `FireKeys` - Open Programs, Folders or URLs with Hot Keys!

[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528)
11 Mar 2013
[CPOL](http://www.codeproject.com/info/cpol10.aspx "The Code Project Open License (CPOL)")12 min read 

FireKeys is a tool that lets you set up hotkey bindings for your favorite programs or places. See how it works, download it and find out how the code was written!

-   [Download FireKeys Source Code - 911.6 KB](https://www.codeproject.com/KB/Tools-IDE/559500/FireKeysSource.zip) 
-   [Install FireKeys (via ClickOnce)](http://www.dwmkerr.com/deploy/firekeys/FireKeys.Application)   

## Introduction   

I don't know when I learnt that Windows + E opened up Windows Explorer. It must have been a while ago. But it's imprinted in my muscle memory, the number of times I hit that combo every day is probably quite high. But how many other hotkeys do I use? Asides from a few other functional ones, like Win + D, I don't use hotkeys so much. And I got to thinking, I'd love to open Google Chrome with a hotkey just like I do with explorer.

So I wrote FireKeys - a lightweight application that lets you assign hotkeys to actions. These actions could be opening program, a folder or a URL, but the underlying model is designed to be extensible. 

![](https://www.codeproject.com/KB/Tools-IDE/559500/FireKeysMain.jpg)

[_Above: A screenshot of FireKeys - a tool to bind actions to Windows Hot Keys_] 

In this article, I'll introduce the tool, show you how it works and how to get it, and then look into some of the more interesting areas of the code that others may be able to benefit from. There'll also be a bit of a discussion on the benefits and drawbacks of ClickOnce deployments.  

## FireKeys  

So let's get started and introduce the FireKeys application. The application can be installed by ClickOnce, just use the link below: 

[Install FireKeys](http://www.dwmkerr.com/deploy/firekeys/FireKeys.Application)  

You can also download the code and build it yourself if you like. Once you install FireKeys, you can open it up by double-clicking the FireKeys tray icon. By default, there will be no bindings set, so the application will be saying that you can open up its suggestions:

![](https://www.codeproject.com/KB/Tools-IDE/559500/MainWindowStart.jpg)

Choose to add Suggestions - the suggestions window will be shown, which offers suggestions for commonly used programs on your machine. 

![](https://www.codeproject.com/KB/Tools-IDE/559500/Suggestions.jpg)

Now that you have some bindings, you can add more with 'New Binding'. You can create a binding to open a program, folder or URL: 

![](https://www.codeproject.com/KB/Tools-IDE/559500/NewHotKeyBinding.jpg)

Finally, the system tray icon for FireKeys will always show what bindings you have set if you click on it. You can open it and manually fire off an action by clicking on its menu item.

![](https://www.codeproject.com/KB/Tools-IDE/559500/TrayIcon.jpg)

## Code Highlights 

Essentially, this program is a very basic WPF application, however, there are some aspects of the program that are interesting to highlight, which may be useful to others creating similar applications. 

1.  [System tray based applications](https://www.codeproject.com/Articles/559500/FireKeys-Open-Programs-Folders-or-URLs-with-Hot-Ke?display=PrintAll#systemtrayapps)
2.  [Using Apex for MVVM](https://www.codeproject.com/Articles/559500/FireKeys-Open-Programs-Folders-or-URLs-with-Hot-Ke?display=PrintAll#mvvm) 
3.  [How Windows Hot Keys work](https://www.codeproject.com/Articles/559500/FireKeys-Open-Programs-Folders-or-URLs-with-Hot-Ke?display=PrintAll#win32hotkeys)  
4.  [Storing data in XML](https://www.codeproject.com/Articles/559500/FireKeys-Open-Programs-Folders-or-URLs-with-Hot-Ke?display=PrintAll#xml) 
5.  [Thinking about the future - plugin models](https://www.codeproject.com/Articles/559500/FireKeys-Open-Programs-Folders-or-URLs-with-Hot-Ke?display=PrintAll#plugins)   

### System Tray Based Applications 

This application is what I would call a 'true' system tray based application. Why is this? Well, from the users point of view, the application runs in the system tray. Opening the exe doesn't seem to do anything apart from put the icon in the system tray and enable the functionality. This is different to an application that you run up which has a main window and a taskbar icon, which _also_ has a system tray icon. This program runs, but in normal usage it has no main window, and in a typical run, doesn't necessarily need to show one.

This has a fundamental affect on the design of the application. Let's take a look at the entry point:

```csharp
 /// <summary>
/// The main entry point for the application.
/// </summary>
[STAThread]
private static void Main(string[] args)
{
    //  Load the arguments.
    var arguments = new Arguments(args);
 
    //  Enable visual styles and set the text rendering modes.
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
 
    //  Load the settings.
    FireKeysApplication.Instance.LoadSettings();
 
    //  Create the context menu.
    CreateContextMenu();
 
    //  Create the tray icon.
    CreateTrayIcon();
 
    //  Create the menu items.
    CreateKeyBindingMenuItems();
 
    //  Whenever the key bindings collection changes, we'll recreate the keybinding menu items.
    FireKeysApplication.Instance.HotKeyBindings.CollectionChanged += (sender, eventArgs) => CreateKeyBindingMenuItems();
    
    //  Run the application.
    Application.Run();
    
    //  Before we terminate, make sure everything's saved.
    FireKeysApplication.Instance.SaveSettings();
}
```

Now we can dissect what's going on easily. 

1.  We load the command line arguments.
2.  We load the application settings. 
3.  If the argument is a specific value, we might immediately show the main window. 
4.  We create a Context Menu (this will be used by the tray icon).
5.  We create the tray icon.
6.  We create the menu items, and when the bindings change we re-create them.
7.  We run the application.

It's important to note that we don't run a main window here, we use Application.Run to keep the application alive, but that's it. This has an important consequence:

**If you have no main Window, integration into Win32 is hard.**

What does this rather obscure sentence mean? Well, imagine we want to register a hotkey binding, no problems, we can interop into the Win32 SDK to do this (and this is detailed later). But what if we want to listen for hotkeys? We need to...... listen for a specific windows message.

**Win32 does most communication via windows messages. So if you don't have a window, you can have a problem.**

Now this is actually quite bad design, in a sense. Sometimes you might want a logical component that can listen for windows messages and run a message pump, but not have any UI. In fact, that's exactly what our application needs?

How do we get around this? Typically, we don't really get around it - we bite the bullet and create the window. Sometimes invisible, other times we hijack another window we need.

In this program, we have to create a context menu object for the system tray icon. The context menu will be created whether or not the user clicks on it, so we use this menu as the object that listens for hot key messages. It's not perfect, and is not ideal design because the context menu logically shouldn't need to do this. It's supposed to be a context menu, not a context menu that listens for hotkeys. However, in the real world we cannot adhere to design patterns perfectly, and should know when to bend the rules. In this case, having the context menu listen for the messages works well.

Now everything else is easy. When the user double clicks on the menu, or activates the 'Settings' item, we can do the following:


```csharp
/// <summary>
/// Shows the settings window.
/// </summary>
private static void ShowSettingsWindow()
{
    //  Is the settings form open? If so, activate it.
    if (mainWindow != null && mainWindow.IsVisible)
    {
        mainWindow.Activate();
        return;
    }
 
    //  Create the settings form.
    mainWindow = new MainWindow();
 
    //  Show the settings form.
    mainWindow.Show();
}
```

If we've already created the settings window, activate it. Otherwise, create it. Easy

If anyone would really like to have a Visual Studio Template for a Context Menu based application, then I'll knock one up. It's been on my todo list for a while, but making Visual Studio templates is slow and fiddly - or give it a bash yourself with the help of [my article on the topic on Item Templates](http://www.codeproject.com/Articles/365680/Extending-Visual-Studio-Part-3-Item-Templates). Project templates are similar. 

If you want to make a WinForms system tray application, you can get the code for [QuickAccent](http://www.codeproject.com/Articles/495239/QuickAccent-A-Tool-for-Accents-and-Symbols) as a starting point - it does the same as the above, but is purely WinForms. 

### Using Apex for MVVM 

Initially I wrote this application with WinForms, thinking it's going to be quick and simple, and that would do the trick. I soon moved to WPF because it's just getting faster and faster to write WPF apps if you have a framework. I use [Apex](http://apex.codeplex.com/ "Apex Framework") for MVVM, it also has a load of useful things like controls, converters and helpers that you sometimes need. I'm not going to go into detail here about Apex, but it is nice to see how it helps with a few common tasks.

#### Easy grids with the `ApexGrid`

The standard WPF grid syntax is too verbose, my grids look like this:

```xml
<apexControls:ApexGrid Rows="Auto,*,Auto"> 
...
</apexControls:ApexGrid>
```

Saves quite a bit of space. Read '[Tidy up XAML with the ApexGrid](http://www.codeproject.com/Articles/233886/Tidy-Up-XAML-with-the-ApexGrid)' for more info. 

Easy form layout with the PaddedGrid

The PaddedGrid builds on the ApexGrid by offering Padding. It's a simple addition, but makes laying out forms easier: 

```xml
<apexControls:PaddedGrid Padding="4" Rows="Auto,Auto" Columns="*,2*">
 
    <Label Grid.Row="0" Grid.Column="0" Content="_Hotkey" />
    <controls:HotKeyControl Grid.Row="0" Grid.Column="1" HotKey="{Binding HotKey}" />
 
    <Label Grid.Row="1" Grid.Column="0" Content="_Display Name" />
    <TextBox Grid.Row="1" Grid.Column="1" Text="{Binding DisplayName}" />
 
</apexControls:PaddedGrid>
```

More detail can be found at '[WPF Padded Grid](http://www.codeproject.com/Articles/107468/WPF-Padded-Grid)'.

**Notifying Properties**

Most MVVM frameworks use a lightweight syntax for defining properties that use INotifyPropertyChanged. I use a slightly more weighty syntax, trying to keep consistent with DepdencyProperties. Here's an example:

```csharp
/// <summary>
/// The NotifyingProperty for the HotKey property.
/// </summary>
private readonly NotifyingProperty HotKeyProperty =
  new NotifyingProperty("HotKey", typeof(HotKey), default(HotKey));
 
/// <summary>
/// Gets or sets HotKey.
/// </summary>
/// <value>The value of HotKey.</value>
public HotKey HotKey
{
    get { return (HotKey)GetValue(HotKeyProperty); }
    set { SetValue(HotKeyProperty, value); }
}
```

It's really quick to write it out with the addition of the Snippets for Apex (installed for Visual Studio as part of the Apex SDK), just key in '`apexnp`' for an apex notifiying property. I have written a whole load of snippets for MVVM: 

1.  `apexnp` - creates an Apex notifying property.

2.  `apexoc` - creates an Observable Collection.

3.  `apexdp` - creates a Dependency Property.

4.  `apexadp` - creates an attached dependency property.

5.  `apexc` - creates an `ICommand` back command.

6.  `apexac` - creates an Apex asynchronous command.

#### File/Folder Text Boxes

It's quite a common task to have a text box with an ellipses to browse for a file. Also common is the same but to browse for a folder. Apex has the FileTextBox and FolderTextBox controls which offer just that. The folder text box pinvokes into the shell for its folder dialog, and doesn't use WinForms.

#### Multi-borders

Again, it's a simple control, a border that lets you specific different brushes, widths etc for each edge, but without it, creating the Wizard style interface is much harder (there are subtle effects like a grey line below each section, with a lighter line to emphasise it). It's the MultiBorder control, there if you need it.

There are a load of other features in Apex I didn't need to use for this app, but for a MVVM application its worth considering. You can use the controls only, or the MVVM stuff, or pick and choose bits and pieces you like. You can get apex from the [Apex Homepage](http://apex.codeplex.com/) or Nuget.  

### How Windows Hotkeys Work

Pretty fundamental to the project is knowing how Windows Hotkeys work. Well, they're straightforward. We ask the Win32 SDK to register a hotkey for us, then listen for the WM_HOTKEY message. Registering a hotkey requires an id, the id of the hotkey used is sent in the windows message. Here's the key MSDN documentation: 

-   [RegisterHotKey](http://msdn.microsoft.com/en-us/library/windows/desktop/ms646309 "RegisterHotKEt") / [UnregisterHotKey](http://msdn.microsoft.com/en-us/library/windows/desktop/ms646327 "Unregister HotKey") 
-   [WM_HOTKEY Message](http://msdn.microsoft.com/en-us/library/windows/desktop/ms646279)  

As mentioned in the first section, you need a message pump to listen for the message - I use the context menu of the application as it is always created. You can pinvoke these functions with the signatures below by the way:

[DllImport("user32.dll")]
private static extern bool RegisterHotKey(IntPtr hWnd, int id, uint fsModifiers, uint vk);
 
[DllImport("user32.dll")]
private static extern bool UnregisterHotKey(IntPtr hWnd, int id);

#### Remember these useful tips as well:

**WinForms 'Keys' enumeration values map to the virtual key codes of the system - case them to uint and you've got the virtual key code.**

**WPF 'Key' enumeration values do NOT map to the virtual key codes, get virtual key codes with the [KeyInterop.VirtualKeyFromKey](http://msdn.microsoft.com/en-us/library/system.windows.input.keyinterop.virtualkeyfromkey.aspx) function!**

`KeyInterop` is provided by the .NET framework for mapping Win32 keys to WPF keys. 

### Storing Data in XML

A first glance at the main program object shows something odd - I save the application settings (which at the moment is just a bool saying whether or not to startup automatically) with an `XmlSerializer`, and the key bindings with a `DataContractSerializer`? Why?

#### `XmlSerializer`

Brilliantly easy to use, lets you choose to store data as attributes or elements, change the name etc. Great for simple data that the end user might theoretically want to look at and change themselves.

#### `DataContractSerializer`

Harder to use, but much more powerful. Allows the serialization of graphs (i.e. interconnected objects) and more complicated entities such as references to interface types.

So why use a DataContractSerializer for my key bindings?

Consider the definition of a key binding below (comments removed for brevity): 

```csharp
[DataContract]
public class HotKeyBinding
{
    /// <summary>    /// Initializes a new instance of the <see cref="HotKeyBinding"/> class.    /// </summary>    public HotKeyBinding()
    {
        IsEnabled = true;
    }
 
    [DataMember]
    public HotKey HotKey { get; set; }
 
    [DataMember]
    public IHotKeyAction Action { get; set; }
 
    [DataMember]
    public string DisplayName { get; set; }
 
    [DataMember]
    public int WindowsHotkeyId { get; set; }
 
    [DataMember]
    public bool IsEnabled { get; set; }
}
```

How the hell can we expect the system to deal with serializing the `IHotKeyAction` typed member 'Action'? It's an interface, how will the framework serialize, and even more difficult, deserialize this to the right type (which might be a `RunProgramAction`, `OpenFolderAction`, `OpenURLAction` etc)?

We the `datacontract` serializer just handles it for you, by writing out the type used in the xml. We must provide the serializer object with the types it'll need to know about (our three interface implementations), but then it'll just work. This is very funky.

This is especially funky when you remember the `DataContractSerializer` is what is used by WCF (and fundamental to it), this sort of behaviour can be extended to work across services. So `DataContractSerializer` is very powerful, but for simple cases, XmlSerializer usually suffices. 

### Plugin Models

I have not had the time to make this application pluggable, but it is only a short way away from being there. Here's what you need to think about when considering it: 

1.  What's a plugin going to need to offer? Keep it simple if possible.
2.  Will it need to present UI? 
3.  How will it deal with logging, exceptions etc?

Typically, consider MEF as a first thing to look at for plugins. I've used it extensively, and it's very powerful. An application like this could be made pluggable very quickly - we've already abstracted out the '`IHotKeyAction`' interface, which represents some kind of action. It already supports custom UI for the action, as well as saving any type derived from `IHotKeyAction` in the key bindings xml file automatically, so we're pretty much there.

If it seems like anyone finds this project or code useful, I'll extend them model to make it pluggable and update the article. 

## `ClickOnce` Deployment 

`ClickOnce` is a pretty nifty way of deploying applications. You publish an application to a share, a website or FTP path, then the user runs a file, such as 'FireKeys.Application'. The system installs the program and runs it. Super easy. You can configure `ClickOnce` to check for updates, not allow the application to run unless it is the latest version and more.

Ever since Microsoft dropped Deployment projects from visual studio (feel free to read my rant on that here ([dwmkerr.com - Deployment Projects in Visual Studio 2012](http://www.dwmkerr.com/2012/12/deployment-projects-in-visual-studio-2012/)), `ClickOnce` is the easiest way to deploy when using Visual Studio 2012. Don't forget the following:

### Pros

-   You can deploy an updated version to a central location and have end users updated automatically.
-   You can force the latest version to be used.
-   You can allow offline mode as well.
-   You can uninstall `ClickOnce` installed programs though the Add/Remove programs applet of Windows.
-   `ClickOnce` is fast, typically a user just hits OK its installed.

### Cons

-   If you deploy to a webserver, and you distribute to a lot of people, you could end up having a lot of traffic to the site.
-   You cannot customise the installation experience.
-   The installed files are deployed to the Users profile, not to 'program files' as many end-users expect.
-   Most anti-virus programs are pretty smart, if they detect things being installed from the web like this, they'll often ask the user to confirm it (in fact, often Windows will too, with its SmartScreen technology). 

As an indicator of how good ClickOnce can be - notice that Google Chrome use it to install their browser! Interestingly enough, I am yet to see many Microsoft based tools that use it, but then again they do tend to make very large heavyweight software that wouldn't be suitable for it. 

## Wrapping Up 

Well that's it. Hopefully, some people will find the tool useful and some will find the code or discussion topics useful. I'm always looking to improve, so if you have any suggestions then let me know!  
  

## License

This article, along with any associated source code and files, is licensed under [The Code Project Open License (CPOL)](http://www.codeproject.com/info/cpol10.aspx)

Written By
**[Dave Kerr](https://www.codeproject.com/Members/DaveKerr)**
Software Developer


## Comments and Discussions

##### My vote of 5
[**D V L**](https://www.codeproject.com/script/Membership/View.aspx?mid=5466984) | 25-Aug-15 0:12 |

nice article  

##### My vote of 5
[Jose David Pujo](https://www.codeproject.com/script/Membership/View.aspx?mid=4621313) | 19-Mar-13 6:34 |

Great Article. Very useful. Thanks!  

##### [Re: My vote of 5](https://www.codeproject.com/Messages/4521370/Re-My-vote-of-5)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 19-Mar-13 7:53 |

Thanks  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)


##### My Vote of 5
[sysko](https://www.codeproject.com/script/Membership/View.aspx?mid=94665) | 12-Mar-13 12:18 |

Forgot to vote in previous question post.  

##### My vote of 5
[Mitchell J.](https://www.codeproject.com/script/Membership/View.aspx?mid=6806656) | 12-Mar-13 6:53 |

Really cool  

##### [Re: My vote of 5](https://www.codeproject.com/Messages/4515470/Re-My-vote-of-5)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 12-Mar-13 9:10 |


Cheers!  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### My vote of 5
[Grasshopper.iics](https://www.codeproject.com/script/Membership/View.aspx?mid=8114613) | 11-Mar-13 23:27 |

AWESOME!  
  
Simple but a very cool tool  

##### [Re: My vote of 5](https://www.codeproject.com/Messages/4515135/Re-My-vote-of-5)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 12-Mar-13 3:38 |


Thanks, the Chrome hotkey is saving me a bit of time, and notepad++  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### Project Load Failure
[sysko](https://www.codeproject.com/script/Membership/View.aspx?mid=94665) | 11-Mar-13 15:23 |

When I try to open the project up, I am getting a failure to load the FireKeys client project. That seems to be missing from the project.  

##### [Re: Project Load Failure](https://www.codeproject.com/Messages/4514907/Re-Project-Load-Failure)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 11-Mar-13 18:04 |


Hi,  
  
Thanks for letting me know! I have just updated the source code in the article, it should work fine now  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### [Re: Project Load Failure](https://www.codeproject.com/Messages/4515640/Re-Project-Load-Failure)
[sysko](https://www.codeproject.com/script/Membership/View.aspx?mid=94665) | 12-Mar-13 12:14 |


That you. That worked.  
  
Compliments on the article. A real trifecta here. Well written. Learned something about Windows programming. And a useful utility to boot. Thanks!  

##### [Re: Project Load Failure](https://www.codeproject.com/Messages/4516082/Re-Project-Load-Failure)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 13-Mar-13 3:49 |


Glad it worked, and thanks for your comments about the article, I'm very pleased it was useful!  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### How to determine existing hot keys?
[Stewart Lindenberger](https://www.codeproject.com/script/Membership/View.aspx?mid=9695765) | 11-Mar-13 15:00 |

I need a way to see a list of the existing hot keys on my W7 x64 system.  
A microsoft utility I've tried to install asks me to choose a hotkey, yet every choice I submit  
comes back as not available. I have asked in other forums how to see which hot keys are active on the system, but no one seems to know the answer.  
  
Can anyone here help?  

##### [Re: How to determine existing hot keys?](https://www.codeproject.com/Messages/4514906/Re-How-to-determine-existing-hot-keys)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 11-Mar-13 18:03 |


Strictly, this is not supported by the Win32 SDK. I've seen some posts from people saying that it can be done by registering low level keyboard hooks and inspecting code around that, but I'm not certain that is a good way of doing it. Short answer seems to be no!  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)


##### Nice Article
[Nicholas Marty](https://www.codeproject.com/script/Membership/View.aspx?mid=8452597) | 11-Mar-13 10:28 |

Thanks for that nice article Some useful stuff in there  
  
If also already tried to register hotkeys and so on. However there was always one last thing which didn't want to work as desired (I can't recall what it was, I guess it was something along overwriting the default hotkeys...). I ended up just scripting everything I wanted in AutoHotkey (which is also very powerful in hooking up Hotkeys).  
  
And since MS dropped the Setup Project in VS2012 I found my self struggling with ClickOnce too (I never fancied that deployment method as it is not neatly packaged into one file and there is a lot of fussing around with certificates and signing (ClickOnce caught me here that it wouldn't launch (while showing no error) when using sha256rsa signature algorithm...

##### [Re: Nice Article](https://www.codeproject.com/Messages/4514905/Re-Nice-Article)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 11-Mar-13 18:00 |

  
Yeah I really wish that with 2012 I had the *choice* of whether I could use ClickOnce or a traditional MSI. Real pain! AutoHotkey looks very impressive btw!  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### My vote of 5
[Ravi Bhavnani](https://www.codeproject.com/script/Membership/View.aspx?mid=191) | 11-Mar-13 9:10 |

Very nice, Dave!  
  
/ravi  

##### [Re: My vote of 5](https://www.codeproject.com/Messages/4514560/Re-My-vote-of-5)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 11-Mar-13 10:48 |


TY Ravi glad you like it  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)

  

##### My vote of 5
[Michael Haephrati](https://www.codeproject.com/script/Membership/View.aspx?mid=5956881) | 10-Mar-13 12:42 |

An excellent article  

##### [Re: My vote of 5](https://www.codeproject.com/Messages/4513924/Re-My-vote-of-5)
[Dave Kerr](https://www.codeproject.com/script/Membership/View.aspx?mid=4259528) | 10-Mar-13 13:32 |


Thanks, glad you like it  

My Blog: [www.dwmkerr.com](http://www.dwmkerr.com/)  
My Charity: [Children's Homes Nepal](http://www.childrenshomesnepal.org/)
