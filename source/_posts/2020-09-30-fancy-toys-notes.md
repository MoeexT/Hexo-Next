---
title: FancyToys 开发笔记
date: 2020-09-30 00:46:31
mathjax: true
categories:
- .Net
tags:
- C#
- UWP
- 笔记
---

FancyToys是一个工具集，里边包含了Nursery。

<!-- more -->

[UWP 大佬](https://www.cnblogs.com/hupo376787/category/1090453.html)

# FancyToys

## 问题

### 同时创建了多个`ContentDialog`

{% note danger %}
System.Exception:“某个异步操作没有正常启动。  Only a single ContentDialog can be open at any time.”
{% endnote %}

``` cs
public static async void Show(string title)
        {
            ContentDialog dialog = new ContentDialog()
            {
                Title = title,
                PrimaryButtonText = "好的",
                CloseButtonText = "Cancel",
                DefaultButton = ContentDialogButton.Primary,
            };
            await dialog.ShowAsync();
        }
```
{% code lang:csharp Utils.cs https://stackoverflow.com/questions/33018346/only-a-single-contentdialog-can-be-open-at-any-time-error-while-opening-anoth StackOverFlow %}
public class DialogUtil
{
    public static ContentDialog ActiveDialog;
    static TaskCompletionSource<bool> DialogAwaiter = new TaskCompletionSource<bool>();
    public static async void CreateContentDialog(ContentDialog Dialog, bool awaitPreviousDialog) { await CreateDialog(Dialog, awaitPreviousDialog); }
    public static async Task CreateContentDialogAsync(ContentDialog Dialog, bool awaitPreviousDialog) { await CreateDialog(Dialog, awaitPreviousDialog); }
    private static void ActiveDialog_Closed(ContentDialog sender, ContentDialogClosedEventArgs args) { DialogAwaiter.SetResult(true); }

    public static async void Info(string text)
    {
        await DialogUtil.CreateContentDialogAsync(new ContentDialog
        {
            Title = "Info",
            Content = new TextBlock
            {
                Text = text,
                TextWrapping = TextWrapping.Wrap
            },
            PrimaryButtonText = "好的"
        }, awaitPreviousDialog: true);
    }
    static async Task CreateDialog(ContentDialog Dialog, bool awaitPreviousDialog)
    {
        if (ActiveDialog != null)
        {
            if (awaitPreviousDialog)
            {
                await DialogAwaiter.Task;
                DialogAwaiter = new TaskCompletionSource<bool>();
            }
            else ActiveDialog.Hide();
        }
        ActiveDialog = Dialog;
        ActiveDialog.Closed += ActiveDialog_Closed;
        await ActiveDialog.ShowAsync();
        ActiveDialog.Closed -= ActiveDialog_Closed;
    }
}
{% endcode %}

### 启动进程失败

+ [Stackoverflow1（未成功）](https://stackoverflow.com/questions/33925096/how-to-execute-process-commands-or-similar-using-a-universal-windows-platform#)
+ [Stackoverflow方式2（有思路）](https://stackoverflow.com/questions/49189353/uwp-how-to-start-an-exe-file-that-is-located-in-specific-directory)


{% note danger %}
System.ComponentModel.Win32Exception:“拒绝访问。”
{% endnote %}

UWP拒绝直接启动进程，只能想办法绕过这个限制；

解决思路：
1. 经[Stackoverflow方式2（有思路）](https://stackoverflow.com/questions/49189353/uwp-how-to-start-an-exe-file-that-is-located-in-specific-directory)的开导，现已经可以用这个方式启动特定进程。但该方式是硬编码，必须在*FancyToys*编译的时候把所需的app的路径加进去，自己用的话还可以接受，但是给别人用的话肯定行不通。此外，启动器打开时有黑框（闪一下），这可不行。
2. 这里边有了app启动器，可以把小黑框去掉，监听*FancyToys*发送的请求，根据请求内容启动相应的app。通信内容包含app路径、运行参数等内容。
3. UWP-客户端，启动器-服务端。做了一个简单的服务器：`NurseryLauncher.exe`，可以接收到start/stop的指令，根据指令做相应操作；启动进程后每隔一段时间向客户端发送进程消息。

### DataGrid的更新

这是最后一个困扰人的问题了


### 托盘图标

上面的不是最后一个，这个可能也不是……
{% note default %}
和NurseryLauncher类似，起一个WinForm进程，用它的托盘图标，同时运行退出就OK
{% endnote %}




## 功能与插件

### 可拖拽区域

+ [UWP拖拽文件](https://www.cnblogs.com/hupo376787/p/8664512.html)
+ [bilibili-添加拖拽](https://www.bilibili.com/video/BV1Ws411y7KW)

和WPF不太一样。与之相比较，出现了两个小问题：
1. 一开始没有实现`DragOver`，误以为不能拖拽，实际上可能已经OK了
2. `Rectangle`没有`Fill`属性的话不能拖拽，但可以设置`Opacity`解决。添加了颜色之后甚至比之前好看。

``` cs
private async void DropArea_Drop(object sender, DragEventArgs e)
{
    var defer = e.GetDeferral();
    try
    {
        DataPackageView dpv = e.DataView;
        if (dpv.Contains(StandardDataFormats.StorageItems))
        {
            List<StorageFile> fileList = new List<StorageFile>();
            var files = await dpv.GetStorageItemsAsync();
            foreach (var item in files)
            {
                DialogUtil.Info(item.Name);
            }
        }
    }
    finally
    {
        defer.Complete();
    }
}
private void DropArea_DragOver(object sender, DragEventArgs e)
{
    e.AcceptedOperation = DataPackageOperation.Copy;
    e.DragUIOverride.Caption = "拖放以添加";
    e.DragUIOverride.IsCaptionVisible = true;
    e.DragUIOverride.IsContentVisible = true;
    e.DragUIOverride.IsGlyphVisible = true;
    e.Handled = true;
}
```

### ToggleSwitchList

当我一开始想做一个ToggleSwitchList的时候，每一个开关都有右键列表，[这里](https://jingyan.baidu.com/article/046a7b3e9abafcf9c27fa9a1.html)给出了一个可行的解决方案，绑定的数据源是`ObservableCollection`。之后发现开关有一个被覆盖的问题：后台代码已经设定好了属性，绑定好了事件，到了这里被模板的`ToggleSwitch`覆盖了，全部变成了xaml设置的样式。可行方案变得不可行。
``` xml
<ListBox x:Name="ProcessListBox" Margin="0,0,0,10" Width="150" Background="{ThemeResource BackgroundAcrylicBrush}" 
    ItemsSource="{Binding SwitchList}" >
    <!-- <ListBox.ItemTemplate>
        <DataTemplate>
            <ToggleSwitch>
                <FlyoutBase.AttachedFlyout>
                    <MenuFlyout>
                        <MenuFlyoutItem/>
                        <MenuFlyoutItem/>
                    </MenuFlyout>
                </FlyoutBase.AttachedFlyout>
            </ToggleSwitch>
        </DataTemplate>
    </ListBox.ItemTemplate> -->
</ListBox>
```
后来就回归原始，不绑定数据源，直接调用`ProcessListBox.Items.Add`方法，删除的时候使用`Remove`方法，问题解决了一半，然后是绑定右键菜单。在页面添加资源字典，ToggleSwitch绑定`RightTapped`事件（一开始在设计页面找的是"RightClick"事件），问题迎刃而解。

``` xml
<Page.Resources>
    <muxc:CommandBarFlyout Placement="Right" x:Name="CommandFlyout" ShowMode="Transient">
        <AppBarButton Label="Args" Icon="Attach" Click="AppBarArgsButton_Click" ToolTipService.ToolTip="添加参数" />
        <AppBarButton Label="Delete" Icon="Delete" Click="AppBarDeleteButton_Click" ToolTipService.ToolTip="删除" />
    </muxc:CommandBarFlyout>
</Page.Resources>
```

### ListBoxItem 高度的设置

一开始在VS属性栏里找不到 ListBoxItem 的设置，后来在 [这里](https://stackoverrun.com/cn/q/10504352)和 [这里](https://stackoverflow.com/questions/49637947/how-to-dynamically-change-the-height-of-listboxitem-uwp)找到了，方法很简单

``` xml
<ListBox x:Name="ProcessListBox" Background="{ThemeResource BackgroundAcrylicBrush}" 
    ItemsSource="{Binding SwitchList}">
    <ListBox.ItemContainerStyle>
        <Style TargetType="ListBoxItem">
            <Setter Property="Height" Value="40" />
            <Setter Property="Padding" Value="10,0,0,0"/>
        </Style>
    </ListBox.ItemContainerStyle>
</ListBox>
```


### C#后台动态修改ListBoxItem属性

我根据xaml尝试了一会没试出来，感谢这位大神 [大牛]([https://link](https://blog.csdn.net/lindexi_gd/article/details/104992276))。

``` cs
private void SetSmallFlyoutItem_Click(object sender, RoutedEventArgs e)
{
    ProcessListBox.ItemContainerStyle = GetStyle(HeightProperty, "32");
}
private Style GetStyle(DependencyProperty property, object value)
{
    Style style = new Style
    {
        TargetType = typeof(ListBoxItem)
    };
    style.Setters.Add(new Setter(property, value));
    style.Setters.Add(new Setter(PaddingProperty, "10,0,0,0"));
    ProcessListBox.ItemContainerStyle = style;
    return style;
}
```
{% note primary %}
注意！在xaml设置的初始属性会被该方法覆盖，例如`ListBoxItem`的padding又恢复默认，需要重新给值。
{% endnote %}
{% img fuckme https://github-pages-1253649638.cos.ap-beijing.myqcloud.com/post-images/2020/UWP%E5%8A%A8%E6%80%81%E6%9B%B4%E6%94%B9ListBoxItem%E9%AB%98%E5%BA%A6.gif 更改ListBoxItem高度 %}

### ContentDialog输入框
来自 [这里](https://jingyan.baidu.com/article/219f4bf7b10624de452d3857.html)
{% note primary %}
解决方案资源管理器可以添加内容对话框（ContentDialog）页面！
{% endnote %}

可能是因为对话框不能直接返回输入值，这里用了公开变量的方法。`isSaved`用于表明是否保存输入，`inputArgs`用于保存输入值。关闭或取消对话框之后，就可以根据上述两个变量进行相应的操作。感谢提出这个方法的作者。
``` cs
public bool isSaved = false;
public string inputArgs = "";
private void ContentDialog_PrimaryButtonClick(ContentDialog sender, ContentDialogButtonClickEventArgs args)
{
    inputArgs = DialogInput.Text;
    isSaved = true;
    Hide();
}

private void ContentDialog_SecondaryButtonClick(ContentDialog sender, ContentDialogButtonClickEventArgs args)
{
    isSaved = false;
    Hide();
}
```
{% img 鸡你太美 https://github-pages-1253649638.cos.ap-beijing.myqcloud.com/post-images/2020/2020-10-04%20ContentDialog.jpg ContentDialog %}

这里还有另一个从StackOverflow找到的Dialog生成方法，链接已不可考……

``` cs
public class DialogUtil
{
    public static ContentDialog ActiveDialog;
    static TaskCompletionSource<bool> DialogAwaiter = new TaskCompletionSource<bool>();
    public static async void CreateContentDialog(ContentDialog Dialog, bool awaitPreviousDialog) { await CreateDialog(Dialog, awaitPreviousDialog); }
    public static async Task CreateContentDialogAsync(ContentDialog Dialog, bool awaitPreviousDialog) { await CreateDialog(Dialog, awaitPreviousDialog); }
    private static void ActiveDialog_Closed(ContentDialog sender, ContentDialogClosedEventArgs args) { DialogAwaiter.SetResult(true); }

    public static async void Info(string text)
    {
        await DialogUtil.CreateContentDialogAsync(new ContentDialog
        {
            Title = "Info",
            Content = new TextBlock
            {
                Text = text,
                TextWrapping = TextWrapping.Wrap
            },
            PrimaryButtonText = "好的"
        }, awaitPreviousDialog: true);
    }
    static async Task CreateDialog(ContentDialog Dialog, bool awaitPreviousDialog)
    {
        if (ActiveDialog != null)
        {
            if (awaitPreviousDialog)
            {
                await DialogAwaiter.Task;
                DialogAwaiter = new TaskCompletionSource<bool>();
            }
            else ActiveDialog.Hide();
        }
        ActiveDialog = Dialog;
        ActiveDialog.Closed += ActiveDialog_Closed;
        await ActiveDialog.ShowAsync();
        ActiveDialog.Closed -= ActiveDialog_Closed;
    }
}
``

### 文件读写

UWP文件读写功能又被限制，一般是只能读写App本地文件夹，还可以请求诸如图片库、视频库等库文件夹，要是想要访问任意文件的话，就需要更改`Package.appxmanifest`了。因为本App需要序列化文件列表及其参数，恰好要用到本地文件读写。具体操作的话可以查看VS的代码提示或 [文档](https://docs.microsoft.com/en-us/windows/uwp/files/quickstart-reading-and-writing-files)。
``` cs
private static StorageFolder localFolder = ApplicationData.Current.LocalFolder;
//读文件
StorageFile file = await localFolder.CreateFileAsync(jsonFile, CreationCollisionOption.OpenIfExists);
string JSONContent = await FileIO.ReadTextAsync(file);
//写文件
string configJson = JsonConvert.SerializeObject(dj);
StorageFile file = await localFolder.CreateFileAsync(jsonFile, CreationCollisionOption.OpenIfExists);
await FileIO.WriteTextAsync(file, configJson);
```

### 服务进程的启动

相关内容请[参见](#启动进程失败)

看`FullTrustProcessLauncher`的文档，有两个方法供启动进程，前者是启动`<desktop:Extension Category="windows.fullTrustProcess" Executable="Assets\Tools\NurseryLauncher.exe">`的，后者是根据`GroupId`启动相应进程的，这里只用前者就够了。

``` cs
// public static IAsyncAction LaunchFullTrustProcessForCurrentAppAsync();
// public static IAsyncAction LaunchFullTrustProcessForCurrentAppAsync(string parameterGroupId);
FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
```




# FancyServer

## 主要工作和项目结构

## 笔记

### switch 判断对象 Type

C#自v7.0支持模式匹配，否则需要一连串的`if else` 用 `is` 关键字判断传入类型。此外，
`typeof`关键字、`object.GetType()`也可以对对象进行类型判断，但不如原生语法灵活。
在[StackOverflow](https://stackoverflow.com/questions/9802325/switch-case-and-generics-checking)
中介绍了其他的类型判断方式。

{% code lang:csharp switch判断传入对象类型 https://docs.microsoft.com/en-us/dotnet/csharp/pattern-matching#conclusions 模式匹配 highlight %}

public static void Send(object sdu)
{
    MessageStruct? pdu = null;
    switch (sdu) 
    {
        case ActionStruct ass:
            pdu = PDU(MessageType.action, JsonConvert.SerializeObject(ass));
            break;
        case LoggingStruct ls:
            pdu = PDU(MessageType.logging, JsonConvert.SerializeObject(ls));
            break;
        case NurseryStruct ns:
            pdu = PDU(MessageType.nursery, JsonConvert.SerializeObject(ns));
            break;
        case SettingStruct ss:
            pdu = PDU(MessageType.setting, JsonConvert.SerializeObject(ss));
            break;
        default:
            LoggingManager.Error("Invalid message SDU type");
            break;
    }
    if (pdu != null)
    {
        PipeMessenger.Post(JsonConvert.SerializeObject(pdu));
    }
}
{% endcode %}

### 修改托盘图标菜单样式
[Stackoverflow](https://stackoverflow.com/questions/15926377/change-the-backcolor-of-the-toolstripseparator-control)

![注意阴影](GrayBackColor.jpg)

大多软件的托盘图标菜单都是纯白的，ContextMenuStrip的默认样式与之相比有点过时，这里需要修改
MenuItem的BackColor属性，但是ToolMenuStripSeparator的BackColor属性在渲染的时候被忽略（或覆盖）了，
需要在separator的Paint事件上加点料：

``` cs
private void mnuToolStripSeparator_Custom_Paint (Object sender, PaintEventArgs e)
{
    ToolStripSeparator sep = (ToolStripSeparator)sender;

    e.Graphics.FillRectangle(new SolidBrush(CUSTOM_COLOR_BACKGROUND), 0, 0, sep.Width, sep.Height);

    e.Graphics.DrawLine(new Pen(CUSTOM_COLOR_FOREGROUND), 30, sep.Height / 2, sep.Width - 4, sep.Height / 2);

}
```

### ContextMenuStrip的部分结构

遍历 `ToolStripMenuItem.DropDownItems` 时要区分 `ToolStripSeparator` 和 `ToolStripDropDownItem` 类型

{% mermaid stateDiagram-v2  %}
ContextMenuStrip --> ToolStripMenuItem
ToolStripMenuItem --> DropDownItems(ToolStripItemCollection)
state DropDownItems(ToolStripItemCollection) {
    ToolStripItem --> ToolStripSeparator
    ToolStripItem --> ToolStripDropDownItem
}
{% endmermaid %}


## Obsolete notes

### 父进程退出后（子进程不退出）端口不释放

{% note info %}
这个问题出自原来的“NurseryLauncher”，现在提供win32服务的是"FancyServer"。与前者相比，
后者结构更加清晰，功能也更加完善。
{% endnote %}

`NurseryLauncher.exe`（绑定626端口）启动 `puppet.exe`（80端口）后，再手动关闭`NurseryLauncher.exe`，
从下图可以看出，根据端口查看进程，有pid存在，但是没有该进程。查子进程`puppet.exe`一切正常。这样导致的问题是：下次`NurseryLauncher.exe`因为端口占用不能正常启动
{% img https://github-pages-1253649638.cos.ap-beijing.myqcloud.com/post-images/2020/2020-10-03%20%E7%88%B6%E8%BF%9B%E7%A8%8B%E9%80%80%E5%87%BA%E4%B8%8D%E9%87%8A%E6%94%BE%E7%AB%AF%E5%8F%A3.jpg  父进程退出但没有释放端口 %}
