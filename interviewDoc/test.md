# 测试

# 一、CodeBlock

```html
<img src="./assets/image-20250311194751130.png" alt="image-20250311194751130" style="zoom:50%;" />
```

# 二、图片(居中)

![image-20250311194751130](./assets/image-20250311194751130.png)

<center>
  <img src="./assets/image-20250311194751130.png" alt="image-2250311194751130" style="zoom:33%;" />
</center>


<img src="./assets/image-20250311194751130.png" alt="image-2250311194751130" style="zoom:33%;" />





# 三、特殊文字（颜色）

<font color="red">这是红色的文字</font>

# 四、特殊符号

、< > 




# 五、mermid
```mermaid
flowchart TD
    Start[开始初始化] --> LoadLayout[加载布局]
    LoadLayout -->|初始化控件| SetupControls[设置控件事件]
    SetupControls -->|初始化音频管理器| AudioManagerSetup[配置音频管理器]
    AudioManagerSetup -->|设置悬浮窗口参数| WindowManagerSetup[配置悬浮窗口]
    WindowManagerSetup -->|启动推流检查| CheckPushMessage[执行推流检查]
    CheckPushMessage --> End[初始化完成]

    subgraph ControlEvents[控件事件处理]
        SwitchCamera[切换摄像头] -->|更新缩放| ResetZoom[重置缩放值]
        VolumeControl[音量控制] -->|切换麦克风状态| UpdateMicState[更新麦克风状态]
        ScreenRecord[录屏控制] -->|判断录屏状态| RecordStateCheck{是否正在录屏}
        RecordStateCheck -->|Yes| StopRecord[停止录屏]
        RecordStateCheck -->|No| ShowToast[提示未录屏]
        WindowSizeChange[窗口大小调整] -->|更新窗口参数| UpdateWindowParams[调整窗口大小和位置]
    end

    Start --> ControlEvents

```

# 六、其他

`加粗` 速度**哈佛**

```java
public static void main(String[] args){
    System.out.prinln("hello world")
}
```

