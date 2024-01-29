<template>
  <div style="height: calc(100vh - constant(safe-area-inset-bottom));height: calc(100vh - env(safe-area-inset-bottom));">
        <section id="importantNote" class="chatContent chat-center" >
            <van-pull-refresh ref="vanPullRefresh" v-model="isRefreshLoading" @refresh="onRefresh" :disabled="isRefreshDisabled">
                <van-list v-model="isListLoading" :finished="isListFinished" finished-text="">
                    <!-- 聊天区 -->
                    <ul>
                        <!-- 历史记录 -->
                        <li class="conversation" v-for="(item,index) in hisList" :key="item.msgId">
                            <div v-if="item.senderRole == 1" class="messageInfo userMessageLi2">
                                <div class="contentCtrol contenForUser">
                                    <p class="content">{{ item.content }}</p>
                                </div>
                                <img class='head'  src="./assets/head_02.png" />
                            </div>
                            <div v-if="item.senderRole == 2 && item.content" class="messageInfo">
                                <img class='head' src="./assets/head_02.png" />
                                <div class="contentCtrol contenForAI">
                                    <div class="content" style="white-space: pre-line;text-align: left;">
                                        <span :id="'hisContentId_'+index" v-html="item.content"></span>
                                        <!-- <el-button :icon="DocumentCopy" class="copy-icon" @click="copyAnswer()" :data-clipboard-target="'#hisContentId_'+index"></el-button> -->
                                    </div>
                                </div>
                            </div>
                        </li>

                        <!-- 提示语 -->
                        <li class="welcome-tips">
                            <p>{{ reminder }}</p>
                        </li>

                        <!-- 对话 -->
                        <li class="conversation" v-for="(item2,index2) in messagesList" :key="index2">
                            <div v-if="item2.senderRole == 1" class="messageInfo userMessageLi2">
                                <div class="contentCtrol contenForUser">
                                    <p class="content">{{ item2.content }}</p>
                                </div>
                                <img class='head'  src="./assets/head_02.png" />
                            </div>
                            <div v-if="item2.senderRole == 2" class="messageInfo">
                                <img class='head' src="./assets/head_02.png" />
                                <div class="contentCtrol contenForAI">
                                    <div class="content" v-if="item2.content=='' && item2.status==0">
                                        <span class="van-loading__spinner van-loading__spinner--spinner" style="color: currentcolor; width: 20px; height: 20px;"><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i><i></i></span>
                                        <span style="display: inline-block;">消息获取中...</span>
                                    </div>
                                    <div class="content" v-if="item2.content == '' && item2.status==2">消息获取失败...<span class="cur" style="color: #0979D7;text-decoration: underline;" @click="_reloadForInputValue(index2)">点击重新获取</span></div>
                                    <div class="content" v-if="item2.content && item2.status==1" style="white-space: pre-line;text-align: left;" >
                                        <div :id="'contentId_'+index2"></div>
                                    </div>
                                </div>
                            </div>
                        </li>
                    </ul>
                </van-list>
               
            </van-pull-refresh>
        </section>
        <section id="chat_footer" class="chatFooter">
                <van-field v-model="userInputValue"  rows="1" type="textarea" enterkeyhint :autosize="{'maxHeight': 100}" placeholder="请输入..." @click-right-icon="_sendForUserInput()">
                    <template #right-icon>
                        <img src="./assets/13.png" style="width: 20px;vertical-align: middle;">
                    </template>
                </van-field>
        </section>
    </div>
</template>

<script>

    import Vue from 'vue';

    import ResizeObserver from 'resize-observer-polyfill';
    import Prism from "./components/prism/prism.js";
    import './components/prism/prism.css';

    import { Cell, List, PullRefresh, ImagePreview, Form, Field } from 'vant';

    Vue.use(Cell);
    Vue.use(List);
    Vue.use(PullRefresh);
    Vue.use(Form);
    Vue.use(Field);
    Vue.use(ImagePreview);

    const robotInformation = {
        "name":"XXX机器人",
        "content":`你好，我是XXX，我很擅长回答XXX相关的一些问题哦，欢迎咨询我~\n我猜你想问：` +
        `<predefined>` +
            `<li class="predefinedLi"  onclick="_sendForPredefinedInput('我是谁？')"><span style="color: #0070c1;text-decoration: underline;cursor: pointer;">我是谁？</span></li>` +
            `<li class="predefinedLi"  onclick="_sendForPredefinedInput('我出生在哪里？')"><span style="color: #0070c1;text-decoration: underline;cursor: pointer;">我出生在哪里？</span></li>` +
            `<li class="predefinedLi"  onclick="_sendForPredefinedInput('有对象吗？')"><span style="color: #0070c1;text-decoration: underline;cursor: pointer;">有对象吗？</span></li>` +
            `<li class="predefinedLi"  onclick="_sendForPredefinedInput('爱我还是ta?')"><span style="color: #0070c1;text-decoration: underline;cursor: pointer;">爱我还是ta?</span></li>` +
        `</predefined>` +
        `点击以上常见问题可直接快速提问。`,
        "tips":"欢迎进入XXX机器人对话页面，机器人旨在更加快速地服务大家，AI生成的内容在小部分场景下可能不一定能非常准确。下拉可加载更多历史记录哦~"
    }

export default {
    name: 'App',
    data() {
        return {
            reminder:'',
            hisList:[],
            messagesList:[],
            isRefreshDisabled:false,
            isRefreshLoading: false,
            isListLoading: false,
            isListFinished: true,
            isLoadingForSystemMessage:false,
            userInputValue:'',
        }
    },
    created() {
        // 加载提醒语
        this.loadReminderMessage();

        // 创建初始对话引导语
        this.loadInitialSession();

        // 需要等待引导语对话框渲染完成后再执行历史记录的加载。保证代码块dom的宽度和引导语dom宽度一致。
        this.$nextTick(function(){ 
            // 加载历史会话
            this.loadHistorySessionList();
        }); 
    },

    mounted(){
        (function() {
            var vendors = ['webkit', 'moz'];
            for (var i = 0; i < vendors.length && !window.requestAnimationFrame; ++i) {
                var vp = vendors[i];
                window.requestAnimationFrame = window[vp+'RequestAnimationFrame'];
                window.cancelAnimationFrame = (window[vp+'CancelAnimationFrame']
                                        || window[vp+'CancelRequestAnimationFrame']);
            }
            if (/iP(ad|hone|od).*OS 6/.test(window.navigator.userAgent) // iOS6 is buggy
                || !window.requestAnimationFrame || !window.cancelAnimationFrame) {
                var lastTime = 0;
                window.requestAnimationFrame = function(callback) {
                    var now = Date.now();
                    var nextTime = Math.max(lastTime + 16, now);
                    return setTimeout(function() {callback(lastTime = nextTime); },nextTime - now);
                };
                window.cancelAnimationFrame = clearTimeout;
            } 
        }());


        window.imgClick = this.imgClick;
        window._sendForPredefinedInput = this._sendForPredefinedInput;


        // 监听footer高度变化
        const observer = new ResizeObserver(this.handleSizeChange); // 创建观察器

        this.$nextTick(() => {
            observer.observe(document.getElementById('chat_footer')); // 指定要观察的DOM元素
            
            // 设置 enterKeyHint 属性值
            document.getElementsByClassName('van-field__control')[0].setAttribute("enterKeyHint", "send"); 
        });
    },
    methods:{
        handleSizeChange(entries) {
            entries.forEach((entry) => {
                document.getElementById('importantNote').style.height = `calc(100% - ${entry.contentRect.height}px)`;
            });
        },
        onRefresh() {
            setTimeout(() => {
                this.isRefreshLoading = false;
            }, 1500);
        },
        loadHistorySessionList(){
            this.isRefreshLoading =true;
            let data = [
                {
                    "senderRole": 1,
                    "name": "user",
                    "content": "图片链接展示",
                },
                {
                    
                    "senderRole": 2,
                    "name": "system",
                    "content":`在互联网上浏览时，我偶然发现了一张非常有趣的图片。这张图片的链接是：https://img95.699pic.com/photo/40250/3647.jpg_wh300.jpg。这张图片展现了一个场景，让人感到既惊喜又温馨。我非常推荐大家也去看看这张图片，相信你也会被它的魅力所吸引。`,
                },
                {

                    "senderRole": 1,
                    "name": "user",
                    "content":"链接地址和代码块展示",
                },
                {
                    
                    "senderRole": 2,
                    "name": "system",
                    "content": `在寻找学习资料时，我找到了一个非常有用的网站链接。这个链接是：https://www.baidu.com。这个网站提供了大量的学习资源，包括视频教程、文章和在线课程等。如果你想学习新的技能或知识，我强烈推荐你访问这个网站，相信你一定会有所收获。
                    另外，以下是代码块，展示了变量命名的重要性：
\`\`\`csharp
// 不好的变量名
int a = 10;
string x = "hello";

// 好的变量名
int age = 10;
string greetingMessage = "hello";
\`\`\``,
                }
            ];
            this.hisList = replaceConvertHTMLToList(data)
            this.isRefreshLoading =false;
        },
        loadReminderMessage(){
            this.reminder = robotInformation.tips
        },

        loadInitialSession(){
            this.messagesList.push({
                "senderRole":2,
                "content":robotInformation.content,
                "contentType":1,
                "contentUrl":null,
                "predefined":robotInformation.predefined,
                "createdTimeValueOf":new Date().valueOf(),
                "createdTime":dateStrFormat(new Date().valueOf()),
                "status":1//消息获取成功
            });
            this.$nextTick(function(){ 
                this.transformTextStream();
            }); 
        },

        _sendForPredefinedInput(value){
            this.sendValue(value,3);
        },

        _sendForUserInput(){
            this.sendValue(this.userInputValue,1);
        },
        _reloadForInputValue(index){
            this.userInputValue='';
            this.sendValue(this.messagesList[index-1].content,2);
        },

        //发送 type 1.发送 2.重试 3.发送预定义内容
        sendValue(inputValue,type){
            this.checkMessage(inputValue);

            this.isLoadingForSystemMessage = true; 

            // 推送用户消息呈现至页面
            this.pushUserMessage(inputValue);

            this.messagesList.push({
                "senderRole":2,
                "content":"",
                "originalContent":"",
                "contentType":1,
                "contentUrl":null,
                "createdTimeValueOf":"",
                "createdTime":"",
                "status":0//消息正在loading
            });
            this.scrollBottom();

            if(type=='1'){
                this.userInputValue = '';
            }

            // 发送会话
            new Promise((resolve,reject)=>{
                setTimeout(() => {
                    let data = 
                [
                    `在互联网上浏览时，我偶然发现了一张非常有趣的图片。这张图片的链接是：https://img95.699pic.com/photo/40250/3647.jpg_wh300.jpg。这张图片展现了一个场景，让人感到既惊喜又温馨。我非常推荐大家也去看看这张图片，相信你也会被它的魅力所吸引。`,
                    `在寻找学习资料时，我找到了一个非常有用的网站链接。这个链接是：https://www.baidu.com。这个网站提供了大量的学习资源，包括视频教程、文章和在线课程等。如果你想学习新的技能或知识，我强烈推荐你访问这个网站，相信你一定会有所收获。`,
                    `以下是代码块，展示了变量命名的重要性：
\`\`\`csharp
// 不好的变量名
int a = 10;
string x = "hello";

// 好的变量名
int age = 10;
string greetingMessage = "hello";
\`\`\`
良好的变量命名能够提高代码的可读性和可维护性。在给变量命名时，应选择有意义的名称，以清晰地表达变量的用途。`,
                ]
                    let number = Math.floor(Math.random() * 4);
                    if(data[number]){
                        resolve(data[number])
                    }else{
                        reject(`获取消息失败，点击重新获取`)
                    }
                    
                }, 1200);
            }).then((res)=>{
                this.isLoadingForSystemMessage = false;
                let obj = this.messagesList[this.messagesList.length-1];
                obj.status = 1;
                obj.content = res;

                this.$set(this.messagesList, this.messagesList.length-1, obj);
                this.$nextTick(function(){ 
                    this.scrollBottom();
                    this.transformTextStream();
                }); 
            }).catch((err)=>{
                this.isLoadingForSystemMessage = false;
                let obj = this.messagesList[this.messagesList.length-1];
                obj.status = 2;

                this.$set(this.messagesList, this.messagesList.length-1, obj);
                this.scrollBottom();
            });
        },
        checkMessage(content){
            if(!content){
                console.log(`请输入内容`);
                return;
            }
            
            if(this.isLoadingForSystemMessage){
                console.log(`请等待上一题回答完成后再提问哦`);
                return;
            }
        },

        pushUserMessage(content){
            let dateValueOf = new Date().valueOf(),
            obj = {
                "senderRole":1,
                "content":content,
                "contentType":1,
                "contentUrl":null,
                "createdTimeValueOf":dateValueOf,
                "createdTime":"",
                "status":1//消息获取成功
            };
            if(dateValueOf - this.messagesList[this.messagesList.length-1].createdTimeValueOf>=30000){
                obj.createdTime = dateStrFormat(dateValueOf);
            }
            this.messagesList.push(obj);
        },

        imgClick(src){
            ImagePreview([src]);
        },


        transformTextStream(){
            (function (that) {
                var container = document.getElementById('contentId_'+ (that.messagesList.length-1));
                container.innerHTML='';
                let {content} = that.messagesList[that.messagesList.length-1];
                let converOBJ;
                let markersList = [];
                let markerHTMLMaps = new Map();

                // 匹配预定义内容
                converOBJ=findPredefinedAndConvertToHTMLTags(content);
                markersList.push(converOBJ.marker);
                markerHTMLMaps.set(converOBJ.marker,converOBJ.arrayHTML)
                content=converOBJ.data;


                //匹配url并替换标签 
                converOBJ=findURLAndConvertToHTMLTags(content);
                markersList.push(converOBJ.marker);
                markerHTMLMaps.set(converOBJ.marker,converOBJ.arrayHTML)
                content=converOBJ.data;

                // 匹配代码块
                converOBJ = find_MDCode_convertTo_HTMLTags(content);
                markersList.push(converOBJ.marker);
                markerHTMLMaps.set(converOBJ.marker,converOBJ.arrayHTML)
                content=converOBJ.data;

                

                //end
                var index = 0;
                function writing() {
                    if (content&&index < content.length) {
                        let dataChild=content[index ++];
                        if(markersList.find((value)=>{return value == dataChild;}) && markerHTMLMaps.get(dataChild).length>0){
                            let arr = markerHTMLMaps.get(dataChild);
                            container.innerHTML +=arr[0].value;
                            //移除前面第1个
                            arr.shift();
                            markerHTMLMaps.set(dataChild,arr);
                        }else{
                            container.innerHTML += dataChild
                        }
                        requestAnimationFrame(writing);
                        document.getElementById('importantNote').scrollTop =9999999999;
                    }
                }
                writing();
            })(this);
        },

        scrollBottom:function(){ 
            this.$nextTick(function(){ 
                document.getElementById('importantNote').scrollTop =9999999999;
            }); 
        },
    }
}

function dateStrFormat(str) {
    var d = new Date(str);
    var h = d.getHours() < 10 ? ('0' + d.getHours()) : d.getHours();
    var m = d.getMinutes() < 10 ? ('0' + d.getMinutes()) : d.getMinutes();
    return d.getMonth() + 1 + "月" + d.getDate() + "日" + " " + h + ":" + m
}

//找到所有预定义标签并转换为HTML标签
function findPredefinedAndConvertToHTMLTags(data){
        // 定义要匹配的模式
    const regex = /<predefined>([\s\S]*?)<\/predefined>/g; // <predefined>...</predefined>形式的HTML标签
    const marker='✿';

    let tempData=data;
    let matchHTML;
    let arrayHTML=[];
    while ((matchHTML = regex.exec(data)) !== null) {
        tempData=tempData.replace(matchHTML[0], marker);
        arrayHTML.push({
            type: `predefined`,
            value: matchHTML[1]
        });
    }

    return {
        marker:marker,
        data:tempData,
        arrayHTML:arrayHTML
    };
}


//找到所有url并转换为HTML标签 
function findURLAndConvertToHTMLTags(data){
    const urlRegex = /http[s]?:\/\/[a-zA-Z0-9$-_@.&+!*(),%0-9a-fA-F]+/g;

    // const imgRegex = /(http[s]?:\/\/[^\s]+\.(jpge|jpg|png|gif|bmp))/g;
    let matchURL;
    let tempData=data;
    let tag= null,arrayHTML=[];
    const marker='۞';
    while ((matchURL = urlRegex.exec(data)) !== null) {
        //不使用变量，不然匹配有问题
        if((/(http[s]?:\/\/[^\s]+\.(jpge|jpg|png|gif|bmp))/g).exec(matchURL[0]) !== null){
        tag = new Object({"type":null,"value":null})
        tag.type = 'img';
        tag.value=`<p style="display: block;" onclick="imgClick('${matchURL[0]}')"><img src="${matchURL[0]}" class="content-img" style="width:100%;" /></p>`
        }else if((/(http[s]?:\/\/[^\s]+\.(mp4|avi|m4v))/g).exec(matchURL[0]) !== null){
        tag = new Object({"type":null,"value":null})
        tag.type = 'video';
        tag.value=`<video controls width="${document.getElementById("contentId_0").offsetWidth}px"><source src="${matchURL[0]}"/></video>`
        }else{
        tag = new Object({"type":null,"value":null})
        tag.type = 'link';
        tag.value=`<a href="${matchURL[0]}" target="_blank" class="content-link" >${matchURL[0]}</a>`
        }
        tempData=tempData.replace(matchURL[0], marker);
        arrayHTML.push(tag);
    }


    
    return {
        marker:marker,
        data:tempData,
        arrayHTML:arrayHTML
    };
}

//找到所有代码块并转换为HTML标签
function find_MDCode_convertTo_HTMLTags(data){
    const marker='☪';
    let tempData=data;
    let arrayHTML=[];
    let match;

    // 匹配代码块
    const regex = /(?:^|\n)(?:\s*\`{3})([a-zA-Z]+)\r?\n((?:.*[\r]?\n)+)(?:^|\n)(?:\s*\`{3})/gm;

    while ((match = regex.exec(data)) !== null) {
        console.log(`匹配到的代码块为:\n${match[0]}`);
        arrayHTML.push({
            type: match[1],/* 代码语言类型 */
            value: `<pre class="language-${match[1]} line-numbers" style="width:${document.getElementById("contentId_0").offsetWidth}px;overflow-x: scroll;box-sizing: border-box;"><code>${Prism.highlight(match[2].trim(), Prism.languages[match[1]], match[1])}</code><pre>`
        });
        tempData=tempData.replace(match[0], marker);
    }

    
    return {
        marker:marker,
        data:tempData,
        arrayHTML:arrayHTML
    };
}

//历史对数据进行匹配
function replaceConvertHTMLToList(list){
    list.forEach(c=>{
        c.createdTime = c.createdTIme?dateStrFormat(new Date(c.createdTIme).valueOf()):'';

        //机器人回答才要转html
        if(c.senderRole == 2){
            let converOBJ;
            let data = c.content;
            let markers = [];
            //匹配url并替换标签  start
            converOBJ=findURLAndConvertToHTMLTags(data);
            if(converOBJ.arrayHTML.length>0){
                for (let i = 0; i < converOBJ.arrayHTML.length; i++) {
                    markers.push({marker:converOBJ.marker,markerHTML:converOBJ.arrayHTML[i]})
                }
            }
            data=converOBJ.data;
            // 匹配代码块
            converOBJ = find_MDCode_convertTo_HTMLTags(converOBJ.data);
            if(converOBJ.arrayHTML.length>0){
                for (let i = 0; i < converOBJ.arrayHTML.length; i++) {
                    markers.push({marker:converOBJ.marker,markerHTML:converOBJ.arrayHTML[i]})
                }
            }
            data=converOBJ.data;
            //end

            
            //替换所有特殊字符
            if(markers.length>0){
                for (let i = 0; i < markers.length; i++) {
                        data=data.replace(markers[i].marker,markers[i].markerHTML.value);
                }
            }
            c.content=data;
        }   
    });
    return list;
}
</script>

<style>
body{height: 100vh;overflow-y: auto;background-color: white;}
.chatContent{
    height: calc(100% - 71px);
    height: calc(100% - 71px );
    height: calc(100% - 71px);
    overflow-y: auto;
}
.chatContent .van-pull-refresh{
    box-sizing: border-box;
}
.van-list{min-height: 80vh;}
.van-list ul{padding:0}

.chat-center {
    box-sizing: border-box;width:100%;
}
.chat-center .welcome-tips{padding: 13px 34px 0;text-align: left;font-size: 12px;line-height: 20px;color: #636666;margin: 0;}
.chat-center .conversation{list-style: none;padding-left: 10px;padding-right: 20px;padding-top: 14px;padding-bottom: 14px;}
.chat-center .predefinedLi{margin-bottom: 6px;line-height: 22px;}
.chat-center .predefinedLi::before{
    content: "";
    width: 4px;
    height: 4px;
    background-color: #0070c1;
    border-radius: 50%;
    display: inline-block;
    margin-right: 4px;
    position: relative;
    top: -2px;
}

.chat-center .messageInfo {
    list-style: none;
    width: 100%;
    display: -webkit-box;
    display: flex;
    align-items: flex-start;
    -webkit-align-items: flex-start;
    text-align: left;
}
.chat-center .messageInfo .contentCtrol {margin: 0 6px;-webkit-flex:1;flex: 1;position: relative;width: 0;clear: both;}
.chat-center .messageInfo .head {width: 44px;height: 44px;border-radius: 50%;}
.chat-center .messageInfo .contentCtrol .title {font-size: 14px;color: gray;}
.chat-center .time {    font-size:12px;line-height:20px;color: #999999;}
.chat-center .messageInfo .contentCtrol .content {
    padding: 10px 15px;
    background-color: #f2f5f7;
    border-radius: 20px;
    font-size: 14px;
    line-height: 26px;
    color:#ffffff;
    -webkit-user-select:text;
    -moz-user-select:text;
    -ms-user-select:text;
    user-select:text;
    -o-user-select:text;
}

.chat-center .messageInfo .contenForUser .content{
    border-top-left-radius: 20px;
    border-top-right-radius: 20px;
    border-bottom-right-radius: 5px;
    border-bottom-left-radius: 20px;
    background-color: #53a8e9;
    float: right;
    color:#ffffff;
    display: inline-block;
    max-width: 100%;
    box-sizing: border-box;
}

.chat-center .messageInfo .contenForAI .content{
    border-top-left-radius: 20px;
    border-top-right-radius: 20px;
    border-bottom-right-radius: 20px;
    border-bottom-left-radius: 5px;
    background-color: #f2f5f7;
    color:#636666;
}
.chat-center .messageInfo .control{
    padding-top: 10px;
    clear: both;
}

.userMessageLi2 {justify-content: flex-end;}


.chatFooter{
    border-top:1px solid #dedede;
}

.chatFooter{box-shadow: 0px 0px 13px 0px rgba(185,185,185,0.41);-webkit-box-shadow: 0px 0px 13px 0px rgba(185,185,185,0.41);}
.chatFooter .van-cell{padding:0 10px}
.chatFooter .van-cell .van-field__body{    border-radius: 18px;background-color: #ececec;padding: 5px 20px;margin: 18px 0;}
.chatFooter .van-field__control{height: 24px;line-height: 24px;}
.chatFooter .van-field__control::placeholder {
  color: #a7a7a7; /* 在此更改占位符文本的颜色 */
}

  .van-empty__image{height: inherit;}
  .tabs_cell .van-tabs__wrap::after{
      border-bottom-color:#c3c3c3
  }
  input[type="search"]{
      font-size: 13px;
  }

  .van-collapse-item__title{
      padding: 10px 14px;
  }

  .van-collapse-item__content{
      padding-top: 0;
  }

  .van-collapse-item__title::after{
      left:0;
      right: 0;
      border-color: #bfbfbf;
  }

  .wprodInfo_cell{
      /* padding:17px 15px;
      padding-right: 10px; */
  }
  .wprodInfo_cell::after{
      left: 0;
      right:0;
      border-color: #bfbfbf;
  }
  .wprodInfo_cell .van-cell__right-icon{
      font-size: 20px;
      color: #000000;
  }
  
  .wprodInfo_title{
      flex: 3;
      text-align: left;
      padding-left: 12px;
      font-size: 13px;
      color: #000000;
  }
</style>
