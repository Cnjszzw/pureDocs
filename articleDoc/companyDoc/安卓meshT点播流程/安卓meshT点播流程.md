# 安卓/meshT点播流程

> 点播设备分三种：安卓/meshT/国标
>
> （1）meshT点播的条件是，自身带视频音频源，或者绑定了国标设备
>
> （2）安卓和meshT底层都用websocket协议和server通信，但是国标是用sip协议

## 1.架构流程图

![image-20250417103651896](assets/image-20250417103651896.png)

![安卓点播流程.drawio](assets/安卓点播流程.drawio.svg)

## 2.点播流程

**前端接口：**

```bash
curl 'http://localhost:6767/api/webrtc/play/video/1/607da3d8526d4d6f837ecc08d5154bd0' \
  -X 'POST' \
  -H 'Accept: application/json, text/plain, */*' \
  -H 'Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6' \
  -H 'Connection: keep-alive' \
  -H 'Content-Length: 0' \
  -b '_ga=GA1.1.2131977798.1744198900; _ga_CCYCB23FSV=GS1.1.1744246933.2.1.1744246937.0.0.0' \
  -H 'Origin: http://localhost:6767' \
  -H 'Sec-Fetch-Dest: empty' \
  -H 'Sec-Fetch-Mode: cors' \
  -H 'Sec-Fetch-Site: same-origin' \
  -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36 Edg/135.0.0.0' \
  -H 'sec-ch-ua: "Microsoft Edge";v="135", "Not-A.Brand";v="8", "Chromium";v="135"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'sec-ch-ua-platform: "Windows"' \
  -H 'token: eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiIxIiwic3ViIjoiYWRtaW4iLCJpc3MiOiJ0b2tlbi1zZXJ2ZXIiLCJpYXQiOjE3NDQ4NTgzMDgsImV4cCI6MTc0NDk0NDcwOCwicm9sZXMiOiJPcGVyYXRpb25Sb2xlLGFkbWluIiwiaWQiOiIxIiwidXNlcm5hbWUiOiJhZG1pbiJ9.4NZ462Me02K59Lybsd-VPvn3et3Rm8Edp7UTeq8En-uK_zJwYx1T9wsh3qDIfatOaqoaUOaXldKwqWBfewAR3A'
```

### 1.进入前端Controller入口

`com/gbcom/wvp/webrtc/controller/WsPlayController.java`，传入用户id和设备(安卓或meshT)

```java
@ApiOperation("申请点播")
    @PostMapping("/video/{demanderId}/{providerId}")
    public DeferredResult<RS> more(@PathVariable String demanderId, @PathVariable String providerId) {
        String eventId = "playVideo" + MessageIdUtil.random();
        DeferredResult<RS> deferredResult = new DeferredResult<>(12000L);
        wsDeferredResultHolder.add(eventId, deferredResult,demanderId);
        playService.playCmd(eventId, demanderId, providerId);
        return deferredResult;
    }
```

### 2.记录点播信息

`com/gbcom/wvp/webrtc/service/impl/WsPlayServiceImpl.java`

```java
@Override
    public void playCmd(String eventId, String demanderId, String providerId) {
        >>---------------------------------------------------
        insertSysCommunicationRecord(1,demanderId,providerId);
		>>---------------------------------------------------
        sendWebsocketMessage(eventId,demanderId,providerId);
    }
```

### 3.发送websocket消息给设备
`com/gbcom/wvp/webrtc/service/impl/WsPlayServiceImpl.java`

```java
@Override
    public void playCmd(String eventId, String demanderId, String providerId) {
        insertSysCommunicationRecord(1,demanderId,providerId);
        >>---------------------------------------------------
        sendWebsocketMessage(eventId,demanderId,providerId);
        >>---------------------------------------------------
    }

private void sendWebsocketMessage(String eventId, String demanderId, String providerId){
        //
        GbUser gbUser1 = new GbUser();
        GbUser gbUser = gbUserMapper.selectByPrimaryKey(demanderId);
        if(gbUser==null){
            gbUser1.setNickname("sup");
            gbUser1.setId(demanderId);
        }else{
            BeanUtils.copyProperties(gbUser,gbUser1);
        }
        JSONObject info = new JSONObject();
        info.put("eventId", eventId);
        info.put("id", demanderId);
        info.put("userName", gbUser1);
        WsSendMessage wsSendMessage = new WsSendMessage(MessageIdUtil.random());
        wsSendMessage.setAction(WsConst.CALL_VIDEO_MSG);
        wsSendMessage.setInfo(info);
        WebsocketSessionManager.send(providerId, wsSendMessage.toJsonString());
    }
```

`com/gbcom/wvp/webrtc/server/WebsocketSessionManager.java`

```java
// 此为单点消息
public static boolean send(Session session, String message) {
    boolean blag = false;
    if (session == null) {
        logger.error("【websocket】 发送消息失败 session == null：" + message + " id:" + getIdBySession(session));
        return blag;
    }
    synchronized (session) {
        if (session.isOpen()) {
            try {
                blag = true;
                session.getBasicRemote().sendText(message);
                logger.info("【websocket】【send->" + getIdBySession(session) + "】" + message);
            } catch (Exception e) {
                blag = false;
                e.printStackTrace();
                logger.error("【websocket】【send Fail->id==>" + getIdBySession(session) + ",type ==>" +sessionMemberType.get(getIdBySession(session))+"】" + message, e);
            }
        }
    }
    return blag;
}
```

### 4.安卓推流

安卓接收到websocket消息，成功后返回ok，并进行推流，推流成功之后**直接上报推流信息**

**请求**

```json
{
    "action": "pushStream",
    "token": "sdfasdfsd",
    "messageId": "随机id标识这个信息",
    "param": {
        "eventId": "点播命令发来的标志",
        "streamInfos": [
            { //”streaminfo” 推流地址 名称 视频 音频 等  待定义
            }
        ]
    }
}
```
**回复**

```json
{
    “code”: 0,
    “message”: “ok”,
	“messageId”: “这个信息的随机id”,
    “data”: {
        result: “ok”,
		reason: 2  // result 不是ok 时，这个请求失败原因  待定义
    }
}
```

以下是相关的代码，底层利用了**`EventBus`** （由 **greenrobot** 开发的事件总线库），订阅了所有的websocket的消息,当**server**服务端监发送了`action`为`CALL_VIDEO_MSG`的websocket消息的时候，安卓客户端就会监听到。

**`java/com/gbcom/rtc/fragment/HomeFragment.java`**

```java
private void initOnMessage() {
        OnDeviceListEventHander.getInstance().setOnDeviceListActivityMessageCallBack(new OnDeviceListEventHander.OnDeviceListActivityMessageCallBack() {
            @Override
            public void onMessageCallback(WsSendMessage wsSendMessage) {

            }

            @Override
            public void onMessageCallback(MessageEvent messageEvent) {
                MessageEventBus(messageEvent);
            }
        });
    }

//@Subscribe(threadMode = ThreadMode.MAIN)
    public void MessageEventBus(MessageEvent messageEvent) {

        switch (messageEvent.getId()) {
            case TV_SHOW://
                //ToastUtils.showToast(messageEvent.getMessage(),mContext);
                Log.d("TAG", messageEvent.getMessage());
                //通过server 端通知设备推流
                String JSONStr = messageEvent.getMessage().split("--")[1];
                WsSendMessage wsSendMessage = JSONObject.parseObject(JSONStr, WsSendMessage.class);
                JSONObject jsonObject = JSONObject.parseObject(JSONStr);
                String action = (String) jsonObject.get("action");
//                const userName = JSON.parse(data).info.userName.username
//                JSON.parse(ev.data).action
                String name = "";
                if (!StringUtils.isEmpty(action)) {
                    switch (wsSendMessage.getAction()) {
                        case WsConst.CALL_VIDEO_MSG:
                            String isHome = findByKey("isHome");
                            if (true) {
                                //通知推流
                                GbUser gbUser = JSONObject.parseObject(String.valueOf(jsonObject.getJSONObject("info").getJSONObject("userName")), GbUser.class);
                                if (!demanderMap.containsKey(gbUser.getId())) {
                                    String demandName = StringUtils.isEmpty(gbUser.getNickname()) ? gbUser.getUsername() : gbUser.getNickname();
                                    demanderMap.put(gbUser.getId(), demandName);
                                }
                                String id = jsonObject.getJSONObject("info").getJSONObject("userName").getString("id");
                                if (!idsList.contains(id)) {
                                    idsList.add(id);
                                }
                                doPush(wsSendMessage);
                            }
                            break;
                        case WsConst.CLOSE_PLAY_VIDEO_MSG:
                            GbUser gbUser = JSONObject.parseObject(String.valueOf(jsonObject.getJSONObject("info").getJSONObject("userName")), GbUser.class);
                            demanderMap.remove(gbUser.getId());
                            if (demanderMap.size() == 0) {
                                stopPush(wsSendMessage);
                            } else {
                                webVideoPopupWindow.changeTitle(handlePlayTitle());
                            }
                            break;
                        case WsConst.CALL_AUDIO_MSG:
                            doPlayWeb(wsSendMessage);
                            break;
                        case WsConst.CLOSE_CALL_AUDIO_MSG:
                            stopPlayWebAudio(wsSendMessage);
                            break;
                        case WsConst.OFFLINE_MSG:
                            //离线
                            //进行check 是否在视频通话
                            WebAccpetPopupWindow webAccpet = (WebAccpetPopupWindow) PopupWindowManager.get("webAccpet");
                            JSONObject jsonInfo = (JSONObject) jsonObject.get("info");
                            String offId = (String) jsonInfo.get("id");
                            String WebVideoTwo = WebAccpetPopupWindow.playUserId;
                            boolean b = offId.equals(WebVideoTwo);
                            if (webAccpet != null) {
                                if (b) {
                                    activity.runOnUiThread(new Runnable() {
                                        @Override
                                        public void run() {
                                            webAccpet.dismiss();
                                        }
                                    });
                                }
                            }
                            WebAudioAccpetPopupWindow webAudioAccpet = (WebAudioAccpetPopupWindow) PopupWindowManager.get("webAudioAccpet");
                            if (webAudioAccpet != null) {
                                if (webAudioAccpet != null && WebAudioAccpetPopupWindow.playUserId.equals(offId)) {
                                    activity.runOnUiThread(new Runnable() {
                                        @Override
                                        public void run() {
                                            webAudioAccpet.dismiss();
                                        }
                                    });
                                }
                            }
                            WebCallPopupWindow webCall = (WebCallPopupWindow) PopupWindowManager.get("webCall");
                            if (webCall != null) {
                                if (webCall != null && WebCallPopupWindow.playMemberId.equals(offId)) {
                                    activity.runOnUiThread(new Runnable() {
                                        @Override
                                        public void run() {
                                            webCall.dismiss();
                                        }
                                    });
                                }
                            }
                            WebAudioCallPopupWindow webAudioCall = (WebAudioCallPopupWindow) PopupWindowManager.get("webAudioCall");
                            if (webAudioCall != null) {
                                if (webAudioCall != null && offId.equals(WebAudioCallPopupWindow.playMemberId)) {
                                    activity.runOnUiThread(new Runnable() {
                                        @Override
                                        public void run() {
                                            webAudioCall.dismiss();
                                        }
                                    });
                                }
                            }
                            WebVideoPopupWindow webVideo = (WebVideoPopupWindow) PopupWindowManager.get("webVideo");

                            if (webVideo != null && WebVideoPopupWindow.playMemberId.equals(offId)) {
                                demanderMap.remove(offId);
                                webVideoPopupWindow.changeTitle(handlePlayTitle());
                                if (demanderMap.size() == 0) {
                                    activity.runOnUiThread(new Runnable() {
                                        @Override
                                        public void run() {
                                            // Update UI elements here
                                            webVideo.dismiss();
                                        }
                                    });
                                }
                            }
                            getMemberList(true);
                            break;
                        case WsConst.ONLINE_MSG:
                            //上线
                            Log.d("ONLINE_MSG", messageEvent.getMessage());
                            getMemberList(true);
                            break;
                        case WsConst.DELETE_MSG:
                            //离线设备删除
                            Log.d("DELETE_MSG", messageEvent.getMessage());
                            getMemberList(true);
                            break;
                        case WsConst.ANDROID_CALL_MSG:
                            Log.d("ANDROID_CALL_MSG", messageEvent.getMessage());
                            //android 视频通话
                            String isHome1 = findByKey("isHome");
                            if (true) {
                                openVideoPopupWindow(wsSendMessage);
                            }
                            break;
                        case WsConst.ANDROID_CLOSE_MSG:
                            Log.d("ANDROID_CLOSE_MSG", messageEvent.getMessage());
                            //android 视频通话
                            CommonUtils.closeVideoPopupWindow(wsSendMessage);
                            break;
                        case WsConst.CONNECT:
                            Log.d("CONNECT", messageEvent.getMessage());
                            //TODD 先不处理心跳
                            //startHeartbeat();
                            break;
                        case WsConst.INVITE_MEETING_ROOM_MSG:
                            Log.d("INVITE_MEETING_ROOM_MSG", messageEvent.getMessage());
                            //doInvite(wsSendMessage);
                            break;
                        case WsConst.CAMERA:
                            //ToastUtils.showToast("收到摄像头消息",mContext);
                            Log.d("CAMERA", messageEvent.getMessage());
                            //摄像头的打开已关闭
                            CommonUtils.checkCamera(wsSendMessage, JSONStr);
                            break;
                        case WsConst.REF_DEVICE:
                        case WsConst.MEMBERINFOCHANGEMSG:
                            Log.d("REF_DEVICE", messageEvent.getMessage());
                            boolean isSelfRefDevice = CommonUtil.checkMessageByMySelf(wsSendMessage,mContext,"member");
                            if(!isSelfRefDevice) {
                                getMemberList(true);
                            }
                            break;
                        case WsConst.OPEN_AUDIO_FOR_SCREEN_MSG:
                            // 如果在视频通话中
                            if (WebCallPopupWindow.play || WebAccpetPopupWindow.play || WebAudioAccpetPopupWindow.play || WebAudioCallPopupWindow.play) {
                                doOpenAudioForScreen(wsSendMessage);
                            }
                            break;
                        case WsConst.CLOSE_AUDIO_FOR_SCREEN_MSG:
                            // 如果在视频通话中
                            if (WebCallPopupWindow.play || WebAccpetPopupWindow.play || WebAudioAccpetPopupWindow.play || WebAudioCallPopupWindow.play) {
                                doCloseAudioForScreen(wsSendMessage);
                            }
                            break;
                        case WsConst.SOS_MSG:
                            showSosMsg(wsSendMessage);
                            break;
                        case WsConst.ENTER_ALARM_AREA_WARNING_MSG:
                            // 进入报警区域
                            //进行一个拦截， 只处理自己的
                            boolean isSelf = CommonUtil.checkMessageByMySelf(wsSendMessage,mContext,"enteredMember");
                            if(isSelf) {
                                showAlarmPopupWindow(wsSendMessage);
                            }
                            break;
                        case WsConst.LEAVE_ALARM_AREA_WARNING_MSG:
                            // 退出报警区域
                            boolean isExitSelf = CommonUtil.checkMessageByMySelf(wsSendMessage,mContext,"leavedMember");
                            if(isExitSelf) {
                                leavedAlarmPopupWindow(wsSendMessage);
                            }
                            break;
                        case WsConst.DELETE_ALARM_AREA_WARNING_MSG:
                            // 删除报警区域
                            refAlarmPopupWindow(wsSendMessage,"true");
                            break;
                        case WsConst.ADD_ALARM_AREA_WARNING_MSG:
                            // 添加报警区域
                            refAlarmPopupWindow(wsSendMessage,"false");
                            break;

                    }
                } else {
                    //先不做处理
                    Log.d("TAG", messageEvent.getMessage());
                }
                break;
            case TV_ERROR:
                Log.d("TAG", messageEvent.getMessage());
                break;
        }
    }
```

上面代码太长了，关键代码如下,主要是接收后端发送过来的websocket消息，记录点播用户的信息

紧接着，判断设备摄像头空闲，立即发送ws消息，先回复ack，

**`java/com/gbcom/rtc/fragment/HomeFragment.java`**

```java
 public void MessageEventBus(MessageEvent messageEvent) {

        switch (messageEvent.getId()) {
            case TV_SHOW://
                //ToastUtils.showToast(messageEvent.getMessage(),mContext);
                Log.d("TAG", messageEvent.getMessage());
                //通过server 端通知设备推流
                String JSONStr = messageEvent.getMessage().split("--")[1];
                WsSendMessage wsSendMessage = JSONObject.parseObject(JSONStr, WsSendMessage.class);
                JSONObject jsonObject = JSONObject.parseObject(JSONStr);
                String action = (String) jsonObject.get("action");
//                const userName = JSON.parse(data).info.userName.username
//                JSON.parse(ev.data).action
                String name = "";
                if (!StringUtils.isEmpty(action)) {
                    switch (wsSendMessage.getAction()) {
                        case WsConst.CALL_VIDEO_MSG:
                            String isHome = findByKey("isHome");
                            if (true) {
                                //通知推流
                                GbUser gbUser = JSONObject.parseObject(String.valueOf(jsonObject.getJSONObject("info").getJSONObject("userName")), GbUser.class);
                                if (!demanderMap.containsKey(gbUser.getId())) {
                                    String demandName = StringUtils.isEmpty(gbUser.getNickname()) ? gbUser.getUsername() : gbUser.getNickname();
                                    demanderMap.put(gbUser.getId(), demandName);
                                }
                                String id = jsonObject.getJSONObject("info").getJSONObject("userName").getString("id");
                                if (!idsList.contains(id)) {
                                    idsList.add(id);
                                }
                                doPush(wsSendMessage);
                            }
                            break;
                            
                            .....
                            .....
                            

private void doPush(WsSendMessage wsSendMessage) {
        //先判断是否在视频通话
        if (WebCallPopupWindow.play || WebAccpetPopupWindow.play || WebAudioAccpetPopupWindow.play || WebAudioCallPopupWindow.play) {
            Log.d("WebVideoTwo", "设备正在通话中,不允许被点播");
            SendMsgUtil.sendAckWsRequestMessage(Objects.requireNonNull(wsSendMessage.getInfo().get("eventId")).toString(), "设备正在视频通话,不允许被点播");
            return;
        }
        //回复
        SendMsgUtil.callBack(wsSendMessage);
        if (PopupWindowManager.get("webVideo") == null) {
            //设置activity 自动横屏
            activity.setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_SENSOR);
            //打开新的播放页面
            String wsStr = JSONArray.toJSONString(wsSendMessage);
            String title = handlePlayTitle();
            activity.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    String isBackground = findByKey("isBackground");
                    if (isBackground.equals("true")) {
                        mTitle = title;
                        mWsstr = wsStr;
                        // 判断手机是否锁屏
                        // 检测是否锁屏
                        checkScreen();
                        // 在这里执行相应的逻辑
                        NotificationUtil.showNotification(mContext, "点播通知", "有人点播了你");
                        Intent intent = new Intent(mContext, HomeActivity.class);
                        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                        mContext.startActivity(intent);
                        VoiceUtils voiceUtils = new VoiceUtils(mContext);
                        voiceUtils.startRingtone();
                    } else {
                        webVideoPopupWindow = new WebVideoPopupWindow((Activity) mContext, title, wsStr, fragment);
                        webVideoPopupWindow.showAtLocation(mRootView,
                                Gravity.TOP | Gravity.LEFT, 0, 0);
                        webVideoPopupWindow.setIdsList(idsList);
                        PopupWindowManager.add("webVideo", webVideoPopupWindow);
                    }
                }
            });
        } else {
            webVideoPopupWindow.changeTitle(handlePlayTitle());
            webVideoPopupWindow.setIdsList(idsList);
            //直接回复消息
            String messageId = wsSendMessage.getMessageId();
            Map<String, Object> info = wsSendMessage.getInfo();
            String eventId = (String) info.get("eventId");
            Map<String, Object> map = new HashMap<>();
            map.put("eventId", eventId);
            zlmHttpsPort = SharedPUtils.findByKey(mContext, "zlmHttpsPort");
            ip = zlmHttpsPort.split(":")[0];
            String playUrl = "http://" + ip + ":6080/index/api/webrtc?app=" + app + "&stream=" + stream + "&type=play";
            String playUrls = "https://" + zlmHttpsPort + "/index/api/webrtc?app=" + app + "&stream=" + stream + "&type=play";
            //String playUrl = "http:"+ip+":6080/index/api/webrtc?app=rtc&stream="+stream+"&type=play";
            StreamInfo streamInfo = new StreamInfo(ip, app, stream, "play", true, true, true, true, true, true, playUrl, playUrls, username, new Date());
            ArrayList<StreamInfo> streamInfoLists = new ArrayList<>();
            streamInfoLists.add(streamInfo);
            map.put("streamInfos", streamInfoLists);
            JSONObject jsonObject = new JSONObject(map);
            SendMsgUtil.callBackWsRequestMessage(WsConst.PUSH_STREAM, messageId, jsonObject, token);
        }
    }                           
```

**`java/com/gbcom/rtc/fragment/HomeFragment.java`**其中,以下为触发代码

```java
if (PopupWindowManager.get("webVideo") == null) {
   //这个if分支为核心分支，为点播流程
   //
    webVideoPopupWindow = new WebVideoPopupWindow((Activity) mContext, title, wsStr, fragment);
        } else {
            //这个分支是已经被点播了，直接返回zlm推流信息即可
        }
```
`com/gbcom/rtc/utils/WebVideoPopupWindow.java`

这段代码是Android应用中实现音视频推流到ZLMediaKit（ZLM）的核心初始化逻辑，主要涉及UI组件、WebRTC渲染、音频管理、用户交互及与ZLM的对接。以下是关键点的详细解释：

------

#### **1. UI组件初始化**

- **布局加载** ：通过`LayoutInflater`加载`web_video_item.xml`布局，初始化视频预览视图（`SurfaceViewRenderer`）、控制按钮（切换摄像头、静音、录屏等）及状态显示组件。
- **控件绑定** ：将布局中的控件（如`btnSmall`、`surfaceViewRenderer1`、`switchCamera`等）与代码中的变量绑定，用于后续操作。

------

#### **2. WebRTC视频渲染配置**

- **EGL上下文初始化** ：创建`EglBase`实例（`mRootEglBase`），为`SurfaceViewRenderer`提供OpenGL上下文，用于视频渲染。

- SurfaceViewRenderer设置 

  ：

  - **渲染回调** ：设置`RendererEvents`监听，处理首帧渲染和分辨率变化。
  - **缩放与镜像** ：使用`SCALE_ASPECT_FILL`模式填充画面，启用硬件加速，关闭镜像（`setMirror(false)`）。

- **摄像头控制** ：通过`webRtcUtil.switchCamera()`切换前后摄像头，`seekBar`调节摄像头缩放级别（调用`webRtcUtil.setCameraZoom()`）。

------

#### **3. 音频管理**

- **音频模式设置** ：将`AudioManager`设为通信模式（`MODE_IN_COMMUNICATION`），优化VoIP场景的音频路由。
- **外放控制** ：根据耳机/蓝牙设备状态动态切换扬声器（`CommonUtil.setAudioSpeakerOn()`），通过按钮切换外放状态。
- **静音与麦克风** ：通过`webRtcUtil.setVoice()`控制麦克风开启/关闭，更新UI状态。

------

#### **4. 用户交互与窗口管理**

- 按钮事件 ：

  - **悬浮窗缩放** ：`btnSmall`触发`changePopupWindowSmall()`调整窗口大小。
  - **录屏控制** ：通过`recordCameraOn/Close`按钮启停录屏服务（`playScreenRecordService`）。
  
- 手势识别 ：

  - **双指缩放** ：通过`ScaleGestureDetector`处理`bigSurface`的缩放手势，动态调整视频画面比例。
- **单击隐藏控件** ：点击`bigSurface`切换控制栏的显示/隐藏。

------

#### **5. 与ZLMediaKit的对接**

- 推流参数准备 ：

  - **服务器地址** ：从SharedPreferences读取`BASE_URl`、`zlmHttpsPort`、`username`等，构建推流地址（如`rtmp://ip:port/app/stream`）。
  - **流名称** ：`stream = username + "Stream"`定义推流的唯一标识。
  
- 信令交互 

  ：

  - **WebSocket消息处理** ：通过`WsSendMessage`解析信令消息，调用`doCheckPushMessage()`处理推流指令（如创建Offer、ICE交换）。
  - **自动横屏** ：首次启动时调用`setRequestedOrientation(true)`强制横屏，适配视频通话场景。

------

#### **6. 关键依赖与未展示逻辑**

- **WebRtcUtil工具类** ：封装WebRTC核心逻辑，如摄像头切换、音量控制、SDP协商等。
- **信令流程** ：实际推流需通过WebSocket与ZLMediaKit交换SDP和ICE候选，建立WebRTC连接（代码中未完全展示）。

------

#### **总结**

这段代码实现了音视频通话的UI交互、WebRTC渲染及基础设备控制，而与ZLMediaKit的推流核心逻辑（如信令交换、流媒体传输）依赖于`webRtcUtil`和后台服务。需结合信令服务器和ZLM的配置，完成完整的推流流程。

```java
private LocalBroadcastManager localBroadcastManager;
    public WebVideoPopupWindow(Activity context, String title, String wsStr, HomeFragment homeFragment) {
        super(context);
        localBroadcastManager = LocalBroadcastManager.getInstance(context);
        //popupWindow = this;
        this.mContext = context;
        this.title = title;
        this.wsStr = wsStr;
        this.homeFragment = homeFragment;
        activity = (Activity) mContext;
        Init();
        wsSendMessageVo = JSONObject.parseObject(wsStr, WsSendMessage.class);
        Map<String, Object> info = wsSendMessageVo.getInfo();
        String demanderId = (String) info.get("id");
        startRingtone();
        play = true;
        OnRequestStreamForRecordEventHander.getInstance().whenOtherStreamChangeEvent();
        playMemberId = demanderId;
        //打开认为 在被点播
        sendDeviceStatus(true);
    }


/**
     * 设置布局以及点击事件
     */
    private void Init() {
        LayoutInflater inflater = (LayoutInflater) mContext
                .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        mView = inflater.inflate(R.layout.web_video_item, null);
        btnSmall = (ImageButton) mView.findViewById(R.id.btn_small);
        surfaceViewRenderer1 = mView.findViewById(R.id.myselfView);
        surfaceViewRendererList.add(surfaceViewRenderer1);
        switchCamera = mView.findViewById(R.id.switchCamera);
        speakClose = mView.findViewById(R.id.speakClose);
        speakOpen = mView.findViewById(R.id.speakOpen);
        muteAudioClose = mView.findViewById(R.id.muteAudioClose);
        muteAudioOpen = mView.findViewById(R.id.muteAudioOpen);
        speakText = mView.findViewById(R.id.speakText);
        muteAudioText = mView.findViewById(R.id.muteAudioText);
        tvTitle = mView.findViewById(R.id.txt_title);
        tvTitle.requestFocus();
        txtTime = mView.findViewById(R.id.txt_time);
        callVideo = mView.findViewById(R.id.btn_callVideo);
        token = SharedPUtils.findByKey(mContext, "token");
        ImageButton btn_closeV = (ImageButton) mView.findViewById(R.id.btn_closeV);
        mRootEglBase = EglBase.create();
        BASE_URl = SharedPUtils.findByKey(mContext, "BASE_URl");
        String gbUserStr = SharedPUtils.findByKey(mContext, "gbUserStr");
        JSONObject jsonObject = JSONObject.parseObject(gbUserStr);
        username = (String) jsonObject.get("username");
        app = (String) jsonObject.get("id");
        zlmHttpsPort = SharedPUtils.findByKey(mContext, "zlmHttpsPort");
        //zlmHttpsPort ="10.30.2.8:10443";
        ip = zlmHttpsPort.split(":")[0];
        //ip="10.30.2.8";
        stream = username + "Stream";
        speedKb = CommonUtil.getSpeedKb(mContext,app);
        topTitle = mView.findViewById(R.id.topTitle);
        callAndAccept = mView.findViewById(R.id.callAndAccept);
        bigSurface = mView.findViewById(R.id.bigSurface);
        recordCameraOn = mView.findViewById(R.id.recordCameraOn);
        recordCameraClose = mView.findViewById(R.id.recordCameraClose);
        recordCameraText = mView.findViewById(R.id.recordCameraText);

        seekBarLinear = mView.findViewById(R.id.seekBarLinear);
        seekBar = mView.findViewById(R.id.seekBar);
        zoomView = mView.findViewById(R.id.zoomView);
        seekBar.setProgress(10);
        zoomView.setText(zoom+"X");
        final Animation clickAnimation = AnimationUtils.loadAnimation(mContext, R.anim.button_click_animation);
        //初始化SurfaceViewRenderer
        for (SurfaceViewRenderer surfaceViewRenderer :
                surfaceViewRendererList) {
            surfaceViewRenderer.init(mRootEglBase.getEglBaseContext(), new RendererCommon.RendererEvents() {
                @Override
                public void onFirstFrameRendered() {

                }

                @Override
                public void onFrameResolutionChanged(int videoWidth, int videoHeight, int rotation) {

                }
            });
            surfaceViewRenderer.setScalingType(RendererCommon.ScalingType.SCALE_ASPECT_FILL);
            surfaceViewRenderer.setEnableHardwareScaler(true);
            surfaceViewRenderer.setZOrderMediaOverlay(false);
            surfaceViewRenderer.setMirror(false);
        }
        if (StringUtils.isEmpty(title)) {
            tvTitle.setVisibility(View.GONE);
        } else {
            tvTitle.setText(title);
        }
        //初始听筒模式
        audioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
        //抢音频焦点
        audioManager.requestAudioFocus(null, AudioManager.STREAM_MUSIC, AudioManager.AUDIOFOCUS_GAIN);
        //有耳机时关闭外放
        if((audioManager.isBluetoothScoAvailableOffCall() && CommonUtil.checkBluetoothConnectionState()) || audioManager.isWiredHeadsetOn()){
            isAudio = false;
            muteAudioClose.setVisibility(View.VISIBLE);
            muteAudioOpen.setVisibility(View.GONE);
            muteAudioText.setText("开启外放");
        }
        if (audioManager != null) {
            audioManager.setMode(AudioManager.MODE_IN_COMMUNICATION);
            CommonUtil.setAudioSpeakerOn(audioManager,isAudio);
//            int streamMaxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
//            audioManager.setStreamVolume(AudioManager.STREAM_MUSIC,streamMaxVolume/2,0);
        }
        // 在初始化阶段设置 touchSlop
        ViewConfiguration vc = ViewConfiguration.get(mContext);
        touchSlop = vc.getScaledTouchSlop();

        // 获取屏幕宽度和高度
        screenWidth = mContext.getResources().getDisplayMetrics().widthPixels;
        screenHeight = mContext.getResources().getDisplayMetrics().heightPixels;

        // 记录PopupWindow的初始大小
        popupWidth = screenWidth;
        popupHeight = screenHeight;
        btnSmall.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //ToastUtils.showToast("123",mContext);
//                popupWindow.setFocusable(false);
//                callVideo.setVisibility(View.VISIBLE);
//                topTitle.setVisibility(View.GONE);
//                callAndAccept.setVisibility(View.GONE);
//                seekBarLinear.setVisibility(View.GONE);
//                imageX = 700;
//                imageY = 300;
//                popupWindow.update(800, 300, 240, 400);
//                activity.setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
                changePopupWindowSmall();
            }
        });
        switchCamera.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.startAnimation(clickAnimation);
                seekBar.setProgress(10);
                zoom=1;
                zoomView.setText(zoom+"X");
                isFrontCamera = !isFrontCamera;
                if(webRtcUtil!=null) {
                    webRtcUtil.switchCamera(isFrontCamera);
                }
            }
        });
        seekBar.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                if (progress < 10) {
                    // 如果是，将其设置为10
                    progress = 10;
                    seekBar.setProgress(10);
                }
                int maxZoom = webRtcUtil.getMaxZoomByIsFrontCamera(isFrontCamera);
                // 更新刻度内容
                zoom = progress * maxZoom / 100;
                if(zoom<1) zoom=1;
                zoomView.setText(zoom+"X");
                webRtcUtil.setCameraZoom(zoom);
            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {
                // 开始拖动时的操作
            }

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {
                // 停止拖动时的操作
            }
        });

        speakClose.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.startAnimation(clickAnimation);
                speakClose.setVisibility(View.GONE);
                speakOpen.setVisibility(View.VISIBLE);
                isOpenSpeak = !isOpenSpeak;
                if(webRtcUtil!=null){
                    webRtcUtil.setVoice(isOpenSpeak);
                }
                speakText.setText("麦克风已开");
            }
        });
        speakOpen.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.startAnimation(clickAnimation);
                speakClose.setVisibility(View.VISIBLE);
                speakOpen.setVisibility(View.GONE);
                isOpenSpeak = !isOpenSpeak;
                if(webRtcUtil!=null){
                    webRtcUtil.setVoice(isOpenSpeak);
                }
                speakText.setText("麦克风已关");
            }
        });
        muteAudioClose.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.startAnimation(clickAnimation);
                muteAudioClose.setVisibility(View.GONE);
                muteAudioOpen.setVisibility(View.VISIBLE);
                isAudio = !isAudio;
                if (audioManager != null) {
                    CommonUtil.setAudioSpeakerOn(audioManager,isAudio);
//                    int streamMaxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
//                    audioManager.setStreamVolume(AudioManager.STREAM_MUSIC,streamMaxVolume,0);
                }
                muteAudioText.setText("关闭外放");
            }
        });
        muteAudioOpen.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                v.startAnimation(clickAnimation);
                muteAudioClose.setVisibility(View.VISIBLE);
                muteAudioOpen.setVisibility(View.GONE);
                isAudio =!isAudio;
                if (audioManager != null) {
                    CommonUtil.setAudioSpeakerOn(audioManager,isAudio);
                }
                muteAudioText.setText("开启外放");
            }
        });
        callVideo.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String isBackground = SharedPUtils.findByKey(mContext,"isBackground");
                if(isBackground=="true") {
                    Intent intent = new Intent(mContext, HomeActivity.class);
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    mContext.startActivity(intent);
                }
                //popupWindow.setFocusable(true);
                // 在这里执行相应的逻辑
                callVideo.setVisibility(View.GONE);
                topTitle.setVisibility(View.VISIBLE);
                callAndAccept.setVisibility(View.VISIBLE);
                seekBarLinear.setVisibility(View.VISIBLE);
                //popupWindow.update(0, 0, popupWidth, popupHeight);
                WindowManager.LayoutParams params = (WindowManager.LayoutParams) mView.getLayoutParams();
                // 缩小窗口
                params.width = WindowManager.LayoutParams.MATCH_PARENT; // 适当的宽度
                params.height = WindowManager.LayoutParams.MATCH_PARENT; // 适当的高度
                params.gravity = Gravity.TOP | Gravity.START;
                params.x = 0;
                params.y = 0;
                // 更新布局参数
                windowManager.updateViewLayout(mView, params);
                //设置activity 自动横屏
                setRequestedOrientation(true);

            }
        });
        bigSurface.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //正在通话
                if (bigFlag) {
                    bigFlag = !bigFlag;
                    callAndAccept.setVisibility(View.GONE);
                    topTitle.setVisibility(View.GONE);
                    seekBarLinear.setVisibility(View.GONE);
                } else {
                    bigFlag = !bigFlag;
                    callAndAccept.setVisibility(View.VISIBLE);
                    topTitle.setVisibility(View.VISIBLE);
                    seekBarLinear.setVisibility(View.VISIBLE);
                }
            }
        });
        recordCameraOn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if(playScreenRecordService != null && !playScreenRecordService.isRunning()){
                    //没有在录屏，无法停止，弹出提示
                    ToastUtils.showToast("您还没有录屏，无法停止，请先开始录屏吧！",mContext);
                }else if(playScreenRecordService != null && playScreenRecordService.isRunning()){
                    v.startAnimation(clickAnimation);
                    recordCameraOn.setVisibility(View.GONE);
                    recordCameraClose.setVisibility(View.VISIBLE);
                    recordCameraText.setText("录像已关");
                    //正在录屏，点击停止，停止录屏
                    playScreenRecordService.stopRecord();
                }

            }
        });
        recordCameraClose.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //screenRecordUtil.requestScreenCapturePermission();
                v.startAnimation(clickAnimation);
                homeFragment.initRecord2();
            }
        });

        windowManager = (WindowManager) mContext.getSystemService(mContext.WINDOW_SERVICE);
        // 设置悬浮窗口的参数
        WindowManager.LayoutParams params = new WindowManager.LayoutParams(
                WindowManager.LayoutParams.MATCH_PARENT,
                WindowManager.LayoutParams.MATCH_PARENT,
                Build.VERSION.SDK_INT >= Build.VERSION_CODES.O ?
                        WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY :
                        WindowManager.LayoutParams.TYPE_PHONE,
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
                PixelFormat.TRANSLUCENT);

        // 设置悬浮窗口的初始位置
        params.gravity = Gravity.TOP | Gravity.START;
        params.x = 0; // 从左边偏移
        params.y = 0; // 从顶部偏移

        // 添加悬浮窗口到 WindowManager
        windowManager.addView(mView, params);
//        // 导入布局
//        popupWindow.setContentView(mView);
//        // 设置动画效果
//        popupWindow.setAnimationStyle(R.style.popwindow_anim_style);
//        popupWindow.setWidth(WindowManager.LayoutParams.MATCH_PARENT);
//        popupWindow.setHeight(WindowManager.LayoutParams.MATCH_PARENT);
//        popupWindow.setFocusable(true);
        mView.findViewById(R.id.btn_callVideo).setFocusable(true);
        // 设置可触
        //ColorDrawable dw = new ColorDrawable(0x0000000);
        //popupWindow.setBackgroundDrawable(dw);
        // 单击弹出窗以外处 关闭弹出窗
        callVideo.setOnTouchListener(new FloatingOnTouchListener());
        WsSendMessage wsSendMessage = JSONObject.parseObject(wsStr, WsSendMessage.class);
        doCheckPushMessage(wsSendMessage);
        // 设置手势缩放监听
        scaleGestureDetector = new ScaleGestureDetector(mContext, new ScaleGestureListener());
        bigSurface.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                if(event.getPointerCount()>1){
                    //处理手指缩放事件
                    return scaleGestureDetector.onTouchEvent(event);
                }else {
                    switch (event.getActionMasked()) {
                        case MotionEvent.ACTION_DOWN:
                            // 处理点击事件
                            v.performClick();
                            break;
                        default:
                            break;
                    }
                    return true;
                }
            }
        });
        surfaceViewRenderer1.setScalingType(RendererCommon.ScalingType.SCALE_ASPECT_FIT);
        //第一次进来就设置为自动横屏
        //设置activity 自动横屏
        setRequestedOrientation(true);
    }
```

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

`com/gbcom/rtc/utils/WebVideoPopupWindow.java`

```java
public void doCheckPushMessage(WsSendMessage wsSendMessage){
        //先去后台把该流清除 ，以防出错
        Api.config(ApiConfig.BASE_URl+ApiConfig.STOPPUSH).postRequestStopPush(app,stream,new rtcCallback() {
            @Override
            public void onSuccess(String res) {
                doWebToAndroidPush(wsSendMessage);
            }

            @Override
            public void onFailure(Exception e) {
                ToastUtils.showToast("网络请求失败", mContext);
            }
        });
    }

private void doWebToAndroidPush(WsSendMessage wsSendMessage) {
        //然后在发起处理
        //"https://10.30.2.8:8970";
        String messageId = wsSendMessage.getMessageId();
        Map<String, Object> info = wsSendMessage.getInfo();
        String demanderId = (String) info.get("id");
        String eventId = (String)info.get("eventId");
        String url ="";
        if(speedKb!=0) {
            url = "http://" + ip + ":6080/index/api/webrtc?app=" + app + "&stream=" + stream + "&type=push"+"&speed="+speedKb;
        }else{
            url = "http://" + ip + ":6080/index/api/webrtc?app=" + app + "&stream=" + stream + "&type=push";
        }
        pushUrlVo = url;
        Log.d("aaaaa", "doWebToAndroidPush: "+webRtcUtil);
        webRtcUtil = WebRtcUtilManager.get("WebToAndroid");
        if(webRtcUtil!=null){
            webRtcUtil.destroy();
        }
        webRtcUtil = new WebRtcUtil(mContext);
        webRtcUtil.setOsdShow(true);

        webRtcUtil.create(isFrontCamera,mRootEglBase, surfaceViewRenderer1, true, true, url, new WebRtcUtil.WebRtcCallBack() {
            @Override
            public void onSuccess() {
                //ToastUtils.showToast("推流成功！",mContext);
                //推流成功之后将消息发送给服务器
                WebRtcUtilManager.add("WebToAndroid", webRtcUtil);
                Map<String,Object> map = new HashMap<>();
                map.put("eventId",eventId);
                String playUrl = "http://"+ip+":6080/index/api/webrtc?app="+app+"&stream="+stream+"&type=play";
                String playUrls = "https://"+zlmHttpsPort+"/index/api/webrtc?app="+app+"&stream="+stream+"&type=play";
                playUrlsVo = playUrls;
                playUrlVo = playUrl;
                //String playUrl = "http:"+ip+":6080/index/api/webrtc?app=rtc&stream="+stream+"&type=play";
                StreamInfo streamInfo = new StreamInfo(ip,app,stream,"play",true,true,true,true,true,true,playUrl,playUrls,username,new Date());
                ArrayList<StreamInfo> streamInfoLists = new ArrayList<>();
                streamInfoLists.add(streamInfo);
                map.put("streamInfos",streamInfoLists);
                JSONObject jsonObject = new JSONObject(map);
                SendMsgUtil.callBackWsRequestMessage(WsConst.PUSH_STREAM,messageId,jsonObject,token);

                lastReloadMeetingTime = System.currentTimeMillis();
                clearIntervalTime();
                mHandler.sendEmptyMessage(0);
                // 当视频通话接通时开始计时
                startTimeMillis = System.currentTimeMillis();
                // 启动计时器
                startTimer();
            }

            @Override
            public void onFail(Exception e) {
                //ToastUtils.showToast("推流失败！",mContext);
                //推流失败的处理
                SendMsgUtil.callBackByAction(WsConst.ACK,messageId,eventId,"设备推流失败了",-1,token);
                closeVideoPopupWindow();
            }
        });
        webRtcUtil.setOnIceConnectionChangeCallBack(new WebRtcUtil.OnIceConnectionChangeCallBack() {
            @Override
            public void onChange(PeerConnection.IceConnectionState state) {
                switch (state) {
                    case DISCONNECTED:
                    case CLOSED:
                        //开启定时任务， 去检测网络是否正常
                        //checkNetWorkTime();
                        break;
                    default:
                        //clearIntervalTime();
                        break;
                }
            }
        });
    }
```

其中doWebToAndroidPush 方法实现了 WebRTC 推流逻辑，主要包括以下步骤：
解析 WsSendMessage 获取推流参数并构造推流 URL。
检查并销毁已存在的 WebRtcUtil 实例，创建新的推流实例。
设置推流回调，成功时发送推流信息到服务器并启动计时器，失败时关闭弹窗并通知服务器。
监听 ICE 连接状态变化，处理网络断开等异常情况。
onSuccess 和 onFail 分别处理推流成功和失败的逻辑，onChange 监听 ICE 状态变化。

```mermaid
flowchart TD
    Start[开始推流] --> Parse[解析消息获取参数]
    Parse --> CheckWebRtc[检查WebRtcUtil实例]
    CheckWebRtc -->|存在| Destroy[销毁旧实例]
    Destroy --> Create[创建新WebRtcUtil实例]
    CheckWebRtc -->|不存在| Create
    Create --> SetCallback[设置推流回调]
    SetCallback --> StartPush[开始推流]
    StartPush --> OnSuccess{推流是否成功}
    OnSuccess -->|Yes| SendInfo[发送推流信息]
    SendInfo --> StartTimer[启动计时器]
    OnSuccess -->|No| HandleFail[处理推流失败]
    HandleFail --> ClosePopup[关闭弹窗]
    StartPush --> MonitorICE[监听ICE状态]
    MonitorICE --> OnChange{ICE状态变化}
    OnChange -->|断开| CheckNetwork[检测网络]
    OnChange -->|其他| Continue[继续]

```

其中具体的推流逻辑如下：

**`com/gbcom/rtc/webrtc/WebRtcUtil.java`**

```java
public void create(Boolean isFrontCamera, EglBase eglBase, SurfaceViewRenderer surfaceViewRenderer, boolean isPublish, boolean isShowCamera, String playUrl, WebRtcCallBack callBack) {
        this.eglBase = eglBase;
        this.surfaceViewRenderer = surfaceViewRenderer;
        this.callBack = callBack;
        this.playUrl = playUrl;
        this.isPublish = isPublish;
        this.isShowCamera = isShowCamera;
        this.isFrontCamera = isFrontCamera;
        init();
    }


public void init() {
        String selfUserId = PreferencesUtil.getString(context, "userId");
        String resolution = PreferencesUtil.getString(context, "meetingVideoSettingResolution" + selfUserId, "640*480");
        VIDEO_RESOLUTION_WIDTH = Integer.parseInt(resolution.split("\\*")[0]);
        VIDEO_RESOLUTION_HEIGHT = Integer.parseInt(resolution.split("\\*")[1]);
        FPS = PreferencesUtil.getInt(context, "meetingVideoSettingFps" + selfUserId, 25);
        peerConnectionFactory = getPeerConnectionFactory(context);
        // NOTE: this _must_ happen while PeerConnectionFactory is alive!
        Logging.enableLogToDebugOutput(Logging.Severity.LS_NONE);

        peerConnection = peerConnectionFactory.createPeerConnection(getConfig(), this);
        MediaConstraints mediaConstraints = new MediaConstraints();

        if (!isPublish) {
            AudioManager audioManager = (AudioManager) context.getSystemService(Context.AUDIO_SERVICE);
            // 将音频路由设置为外放
            ///audioManager.setMode(AudioManager.MODE_NORMAL);
            //CommonUtil.setAudioSpeakerOn(audioManager,true);
            //设置仅接收音视频
            peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_AUDIO, new RtpTransceiver.RtpTransceiverInit(RtpTransceiver.RtpTransceiverDirection.RECV_ONLY));
            peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_VIDEO, new RtpTransceiver.RtpTransceiverInit(RtpTransceiver.RtpTransceiverDirection.RECV_ONLY));
        } else {
            //设置仅推送音视频
            peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_AUDIO, new RtpTransceiver.RtpTransceiverInit(RtpTransceiver.RtpTransceiverDirection.SEND_ONLY));
            peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_VIDEO, new RtpTransceiver.RtpTransceiverInit(RtpTransceiver.RtpTransceiverDirection.SEND_ONLY));

            //设置回声去噪
            WebRtcAudioUtils.setWebRtcBasedAcousticEchoCanceler(true);
            WebRtcAudioUtils.setWebRtcBasedNoiseSuppressor(true);
//            WebRtcAudioUtils.setDefaultSampleRateHz(8000);

            // 音频
            audioSource = peerConnectionFactory.createAudioSource(createAudioConstraints());
            audioTrack = peerConnectionFactory.createAudioTrack(AUDIO_TRACK_ID, audioSource);
            audioTrack.setVolume(8.0f);
            audioTrack.setEnabled(true);

            peerConnection.addTrack(audioTrack);
            //是否显示摄像头画面
            //if (isShowCamera) {
            videoCapturer = CameraUtil.createCameraVideoCapturer(context, isFrontCamera);
            surfaceTextureHelper = SurfaceTextureHelper.create("CaptureThread", eglBase.getEglBaseContext());
            videoSource = peerConnectionFactory.createVideoSource(false);
            //videoSource.setVideoProcessor(videoProcessor);
            videoCapturer.initialize(surfaceTextureHelper, context, videoSource.getCapturerObserver());
            videoCapturer.startCapture(VIDEO_RESOLUTION_WIDTH, VIDEO_RESOLUTION_HEIGHT, FPS);
            videoTrack = peerConnectionFactory.createVideoTrack(VIDEO_TRACK_ID, videoSource);
            videoTrack.setEnabled(true);
            if (surfaceViewRenderer != null && isShowCamera) {
                videoSink = new ProxyVideoSink(this.eglBase.getEglBaseContext());
                setRenderScalingType();
                videoSink.setTarget(surfaceViewRenderer);

                boolean meetingTimeWatermark = PreferencesUtil.getBoolean(MyApplication.getAppContext(), "meetingTimeWatermark" + selfUserId, true);
                int meetingTimeWatermarkPosition = PreferencesUtil.getInt(MyApplication.getAppContext(), "meetingTimeWatermarkPosition" + selfUserId, 2);
                boolean meetingTitleWatermark = PreferencesUtil.getBoolean(MyApplication.getAppContext(), "meetingTitleWatermark" + selfUserId, true);
                int meetingTitleWatermarkPosition = PreferencesUtil.getInt(MyApplication.getAppContext(), "meetingTitleWatermarkPosition" + selfUserId, 1);
                String meetingTitleText = PreferencesUtil.getString(MyApplication.getAppContext(), "meetingTitleText" + selfUserId, null);
                int meetingWatermarkMargin = Integer.parseInt(PreferencesUtil.getString(MyApplication.getAppContext(), "meetingWatermarkMargin" + selfUserId, "25"));
                int meetingWatermarkTextsize = Integer.parseInt(PreferencesUtil.getString(MyApplication.getAppContext(), "meetingWatermarkTextSize" + selfUserId, "10"));
                if (osdShow && (meetingTimeWatermark || meetingTitleWatermark) && videoSource != null && watermarkVideoProcessor == null) {
                    watermarkVideoProcessor = new WatermarkVideoProcessor(this.surfaceTextureHelper);
                    watermarkVideoProcessor.setSink(videoSink);
                    watermarkVideoProcessor.setHasTimestamp(meetingTimeWatermark);
                    watermarkVideoProcessor.setTimeWatermarkPosition(meetingTimeWatermarkPosition);
                    watermarkVideoProcessor.setHasTitle(meetingTitleWatermark);
                    watermarkVideoProcessor.setTitleWatermarkPosition(meetingTitleWatermarkPosition);
                    watermarkVideoProcessor.setTitle(meetingTitleText);
                    watermarkVideoProcessor.setDefaultMargin(meetingWatermarkMargin);
                    watermarkVideoProcessor.setTextSize(meetingWatermarkTextsize);
                    videoSource.setVideoProcessor(watermarkVideoProcessor);
                }

                videoTrack.addSink(videoSink);
            }
            lastRtpSender = peerConnection.addTrack(videoTrack);
            //}
        }
        peerConnection.createOffer(this, mediaConstraints);
    }
```

```mermaid
flowchart TD
    Start[开始初始化] --> ReadConfig[读取用户配置]
    ReadConfig --> CreateFactory[创建PeerConnectionFactory]
    CreateFactory --> CreateConnection[创建PeerConnection]
    CreateConnection --> CheckMode{判断是否为推流模式}
    CheckMode -->|否| RecvOnly[配置接收音视频]
    CheckMode -->|是| SendOnly[配置推送音视频]
    SendOnly --> InitAudio[初始化音频源]
    InitAudio --> InitVideo[初始化视频源]
    InitVideo --> AddWatermark{是否需要添加水印}
    AddWatermark -->|是| ProcessWatermark[配置水印处理器]
    AddWatermark -->|否| SkipWatermark[跳过水印处理]
    SkipWatermark --> AddTrack[添加音视频轨道]
    ProcessWatermark --> AddTrack
    AddTrack --> CreateOffer[创建SDP Offer]
    RecvOnly --> CreateOffer
    CreateOffer --> End[完成初始化]

```

这段代码是基于 **WebRTC** 实现音视频推流（或接收）的核心初始化逻辑，主要用于配置音视频参数、创建 PeerConnection、管理音视频轨道，并处理与 ZLMediaKit 服务器的连接。以下是对代码的详细解释：

#### **1. 参数初始化**

```java
String selfUserId = PreferencesUtil.getString(context, "userId");
String resolution = PreferencesUtil.getString(context, "meetingVideoSettingResolution" + selfUserId, "640*480");
VIDEO_RESOLUTION_WIDTH = Integer.parseInt(resolution.split("\\*")[0]);
VIDEO_RESOLUTION_HEIGHT = Integer.parseInt(resolution.split("\\*")[1]);
FPS = PreferencesUtil.getInt(context, "meetingVideoSettingFps" + selfUserId, 25);
```

- 功能 ：从本地存储中读取用户配置的音视频参数：
  - `selfUserId`：用户唯一标识。
  - `resolution`：视频分辨率（默认 `640*480`）。
  - `FPS`：视频帧率（默认 25 帧/秒）。
- **用途** ：这些参数后续用于配置摄像头采集和编码。

------

#### **2. PeerConnectionFactory 和 PeerConnection 创建**

```java
peerConnectionFactory = getPeerConnectionFactory(context);
peerConnection = peerConnectionFactory.createPeerConnection(getConfig(), this);
```

#### **3. 音视频传输方向配置**

```java
if (!isPublish) {
    // 接收端：仅接收音视频
    peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_AUDIO, RECV_ONLY);
    peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_VIDEO, RECV_ONLY);
} else {
    // 推流端：仅发送音视频
    peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_AUDIO, SEND_ONLY);
    peerConnection.addTransceiver(MediaStreamTrack.MediaType.MEDIA_TYPE_VIDEO, SEND_ONLY);
}
```

- Transceiver 模式 ：
  - `RECV_ONLY`：接收端（如观看直播），仅接收远端音视频。
  - `SEND_ONLY`：推流端（如摄像头直播），仅发送本地音视频到 ZLMediaKit。

#### **4. 音频处理**

```java
// 启用 WebRTC 内置的回声消除和降噪
WebRtcAudioUtils.setWebRtcBasedAcousticEchoCanceler(true);
WebRtcAudioUtils.setWebRtcBasedNoiseSuppressor(true);

// 创建音频轨道
audioSource = peerConnectionFactory.createAudioSource(createAudioConstraints());
audioTrack = peerConnectionFactory.createAudioTrack(AUDIO_TRACK_ID, audioSource);
audioTrack.setVolume(8.0f); // 设置音量增益
peerConnection.addTrack(audioTrack);
```

- **音频优化** ：启用回声消除（AEC）和降噪（NS），提升通话质量。
- **音频轨道** ：通过 `AudioSource` 采集麦克风数据，封装为 `AudioTrack` 并添加到 `PeerConnection`。

#### **5. 视频处理**

##### **摄像头初始化**

```java
videoCapturer = CameraUtil.createCameraVideoCapturer(context, isFrontCamera);
surfaceTextureHelper = SurfaceTextureHelper.create("CaptureThread", eglBase.getEglBaseContext());
videoSource = peerConnectionFactory.createVideoSource(false);
videoCapturer.initialize(surfaceTextureHelper, context, videoSource.getCapturerObserver());
videoCapturer.startCapture(VIDEO_RESOLUTION_WIDTH, VIDEO_RESOLUTION_HEIGHT, FPS);
```

- **摄像头选择** ：通过 `CameraUtil` 创建摄像头捕获器（前置/后置）。
- **视频源** ：`videoSource` 接收摄像头数据，绑定到 `SurfaceTextureHelper` 管理 OpenGL 渲染线程。
- **启动采集** ：设置分辨率、帧率，开始摄像头数据捕获。

##### **视频轨道与渲染**

```java
videoTrack = peerConnectionFactory.createVideoTrack(VIDEO_TRACK_ID, videoSource);
videoTrack.addSink(videoSink); // 绑定渲染视图
peerConnection.addTrack(videoTrack);
```

- **视频轨道** ：将摄像头数据封装为 `VideoTrack`，添加到 `PeerConnection`。
- **渲染** ：通过 `ProxyVideoSink` 将视频帧输出到 `SurfaceViewRenderer`，实现实时预览。

#### **6. 水印叠加（Watermark）**

```java
watermarkVideoProcessor = new WatermarkVideoProcessor(surfaceTextureHelper);
watermarkVideoProcessor.setSink(videoSink);
watermarkVideoProcessor.setHasTimestamp(meetingTimeWatermark);
watermarkVideoProcessor.setTitle(meetingTitleText);
videoSource.setVideoProcessor(watermarkVideoProcessor);
```

####  **7.创建 Offer 并开始推流**

```java
peerConnection.createOffer(this, mediaConstraints);
```

- **SDP 协商** ：生成本地会话描述（Offer），通过信令服务器发送到 ZLMediaKit，完成媒体协商。
- **后续流程** ：需处理 `onCreateSuccess` 回调，设置本地描述，并交换 ICE 候选以建立连接。

------

#### **关键点总结**

1. **推流模式** ：通过 `isPublish` 区分推流端/接收端，配置不同的 Transceiver 方向。
2. **音视频优化** ：启用 WebRTC 内置的 AEC/NS，提升通信质量。
3. **动态配置** ：分辨率、帧率、水印等参数通过 `SharedPreferences` 动态读取，支持用户自定义。
4. **与 ZLMediaKit 对接** ：通过 `PeerConnection` 的 SDP 和 ICE 交互，将音视频流推送到 ZLMediaKit 服务器。

------

#### **潜在扩展点**

- **错误处理** ：需添加对摄像头、麦克风权限的检查，以及 PeerConnection 创建失败的重试逻辑。
- **资源释放** ：在 `onDestroy` 中释放 `videoCapturer`、`surfaceTextureHelper` 等资源。
- **自适应码率** ：根据网络状况动态调整分辨率和码率（需结合 WebRTC 的带宽估计模块）。

这段代码是音视频推流的核心逻辑，需结合信令服务器（如 WebSocket）和 ZLMediaKit 的配置，完成完整的媒体传输流程

### 5.获取推流信息，进行拉流

服务端对PUSH_STREAM消息进行监听，收到安卓的推流推送消息，进行出来，setResult后结束上轮询，上报前端推流信息

`com/gbcom/wvp/webrtc/processor/request/imp/PushStreamRequestProcessor.java`

```java
@Component
public class PushStreamRequestProcessor extends RequestProcessorParent implements InitializingBean, IRequestProcessor {
    private final Logger losgger = LoggerFactory.getLogger(PushStreamRequestProcessor.class);

    @Autowired
    private WebSocketProcessorObserver webSocketProcessorObserver;

    @Autowired
    private MemberService memberService;

    @Autowired
    private WsDeferredResultHolder wsDeferredResultHolder;

    @Autowired
    private WebSocketClient webSocketClient;

    @Autowired
    private MeshSysConfigService meshSysConfigService;

    @Autowired
    private ZLMRESTfulUtils zlmresTfulUtils;

    @Override
    public void afterPropertiesSet() throws Exception {
        // 添加消息处理的订阅
        webSocketProcessorObserver.addRequestProcessor(WsConst.PUSH_STREAM, this);
    }

    @Override
    public void process(Session session, WsRequestMessage message) {
        System.out.println("--11111111111111111111111111");
        WsResponseMessage responseMessage = new WsResponseMessage(message.getMessageId());
        responseMessage.resultOk();

        JSONObject param = message.getParam();
        JSONArray streamInfos = new JSONArray();
        String eventId = null;
        if (param.containsKey("eventId")) {
            eventId = param.getString("eventId");
        }
        if (param.containsKey("streamInfos")) {
            streamInfos.addAll(param.getJSONArray("streamInfos"));
        }
        if (streamInfos.size() == 0) {
            responseMessage.resultNotOk().addData("reason", 10).addData("info", "空数据");
        }
        if (StringUtils.isBlank(eventId)) {
            responseMessage.resultNotOk().addData("reason", 10).addData("info", "无eventId");
        }
        WebsocketSessionManager.send(session, responseMessage);
        if (!responseMessage.resultIsOk()) {
            return;
        }
        String id = WebsocketSessionManager.getIdBySession(session);
        for (Object streamInfo : streamInfos) {
            JSONObject object = (JSONObject) streamInfo;
            StreamInfo info = JSON.parseObject(JSON.toJSONString(object), StreamInfo.class);
            memberService.saveOrUpdateMemberStreamInfo(id, info);
        }
        RS rs = new RS();
        System.out.println("--2222222222222222222");
        rs.ok().addData("streamInfos", streamInfos);
        wsDeferredResultHolder.complete(eventId, rs,id);
        System.out.println("--3333333333333333333333");
        // 处理级联的点播消息
        if(eventId.endsWith("ForCascade")){
            eventId = eventId.replace("ForCascade","");
            Map<String, Object> map = meshSysConfigService.cascadeParam();
            if(map!=null && (Boolean) map.get("isOpen")){
                String url = (String) map.get("url");
                // 将流转推到上级zlm
                JSONObject obj = (JSONObject) streamInfos.get(0);
                StreamInfo info = JSON.parseObject(JSON.toJSONString(obj), StreamInfo.class);
                String schema = "rtsp";
                String app = info.getApp();
                String stream = info.getStream();
                String dstUrl = schema+"://"+url+":10554/"+app+"/"+stream;
                // 推流代理
                Map<String, Object> params = new HashMap<>();
                params.put("vhost", "__defaultVhost__");
                params.put("schema", schema);
                params.put("app", app);
                params.put("stream",stream);
                params.put("dst_url",dstUrl);
                // 延时1秒 避免流还未推
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                com.alibaba.fastjson2.JSONObject jsonObject = zlmresTfulUtils.addStreamPusherProxy(params);
                if(jsonObject.getInteger("code").equals(0)){
                    // 给上级websocket也发送此消息
                    logger.info("多级级联点播，发送上级消息：{}",message);
                    webSocketClient.sendMessage(JSON.toJSONString(message));
                }
            }
            wsDeferredResultHolder.complete(eventId, rs,id);
        }
    }
}
```



## Q&A

### 1.Redis在其中的作用到底是什么？

代码中没有用到好像，只是存起来了。

### 2.websocket消息的订阅是如何实现的？

### 3.服务器是一个什么概念到底？就是websocket？可是后者不是一个网络协议吗

信令服务器是音视频通信中的一个核心概念，但它的实现方式可以多种多样（比如 WebSocket、HTTP、TCP/UDP 自定义协议等）。以下是详细解释：

------

#### **1. 信令服务器的核心作用**

信令服务器是 **协调通信双方建立连接的中间服务** ，主要功能包括：

- **交换元数据** ：如 SDP（媒体能力描述）、ICE 候选（网络地址信息）。
- **控制通信流程** ：如发起/结束通话、协商编解码器、处理错误。
- **用户状态管理** ：如在线状态、房间管理（多人会议场景）。

在 WebRTC 中，信令服务器是 **必须的** ，但 WebRTC 协议本身 **不规定信令的具体实现方式** ，开发者可以自由选择协议。

------

#### **2. WebSocket 是什么？**

WebSocket 是一种 **基于 TCP 的全双工通信协议** ，特点包括：

- **长连接** ：客户端和服务器建立连接后，双方可随时主动发送数据。
- **低延迟** ：避免了 HTTP 请求/响应的握手开销。
- **适合实时场景** ：如聊天、实时数据推送、信令传输。

------

#### **3. 信令服务器与 WebSocket 的关系**

- **信令服务器是一个逻辑概念** ：它负责处理通信双方的协调逻辑。

- WebSocket 是实现信令服务器的一种常见手段 ：

  - 通过 WebSocket，客户端和服务器可以实时交换 SDP 和 ICE 信息。
- 例如：在 WebRTC 中，客户端通过 WebSocket 向信令服务器发送 SDP Offer，服务器转发给远端，远端回复 SDP Answer。

------

#### **4. 为什么信令服务器常使用 WebSocket？**

- **实时性** ：WebSocket 的全双工特性天然适合实时信令传输。
- **简单易用** ：相比自定义 TCP/UDP 协议，WebSocket 有成熟的库和框架支持。
- **跨平台兼容性** ：浏览器和移动端均可方便地实现 WebSocket 客户端。

------

#### **5. 其他信令实现方式**

信令服务器不局限于 WebSocket，还可以用：

- **HTTP 长轮询** ：客户端定期向服务器请求更新（如早期 Web 应用）。
- **SIP 协议** ：传统 VoIP 系统常用（如 Asterisk）。
- **MQTT** ：适用于物联网场景的轻量级消息协议。
- **自定义 TCP/UDP 协议** ：对性能或安全性有特殊要求时。

------

**6. 结合 ZLMediaKit 的实际场景**

在你的代码中，信令服务器可能通过 WebSocket 与 ZLMediaKit 交互：

1. **推流端** ：通过 WebSocket 发送 SDP Offer 到信令服务器。
2. **信令服务器** ：将 Offer 转发给 ZLMediaKit，接收其生成的 SDP Answer。
3. **ZLMediaKit** ：处理媒体流传输（如 RTMP、WebRTC）。

------

#### **总结**

- **信令服务器** ：逻辑概念，负责协调通信双方。
- **WebSocket** ：一种实现信令传输的协议（全双工、低延迟）。
- **关系** ：WebSocket 是信令服务器的常见实现方式，但非唯一选择。

如果需要进一步了解信令交互的具体流程（如 SDP 交换、ICE 候选传递），可以结合你的代码中的 `WsSendMessage` 和 `doCheckPushMessage` 方法分析！

### 4.**Q :** 为什么要用synchronized

- - **WebSocket 的多线程场景** ：

    - WebSocket 服务器通常基于异步、多线程模型处理请求。例如，一个客户端可能同时触发多个事件（如消息发送、心跳检测、会话关闭等），这些操作可能由不同的线程并发执行。

    - 如果多个线程同时调用 

      ```
      session.getBasicRemote().sendText()
      ```

       向同一个会话发送消息，可能导致以下问题：

      - **消息乱序** ：多个线程同时写入网络流，导致消息顺序错乱。
      - **会话状态不一致** ：在发送消息前检查 `session.isOpen()` 后，另一个线程可能突然关闭会话，导致后续操作失败。
      - **资源竞争** ：底层网络资源（如 TCP 连接）可能被多个线程同时修改，引发不可预知的异常。

- - **通过 `synchronized` 同步** ：
- 使用 `synchronized(session)` 可确保同一时间只有一个线程能进入该代码块，对 `session` 的操作（检查状态、发送消息）是原子性的，避免了并发问题。

### 5.**Q:** websocket是怎么用netty的，我看项目中有用netty



