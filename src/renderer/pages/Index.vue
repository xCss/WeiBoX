<template>
    <div class="home" v-loading="loading" :element-loading-text="loadingText" >
        <div class="drag-wrap">
            点击或拖拽图片到此
        </div>
        <div class="content-wrap">
            <el-row class="options" :gutter="20" :type="'flex'" align="middle">
                <el-col :span="8" style="text-align:center;">
                    <el-button-group>
                        <el-button :type="size === 'thumbnail' ? 'primary':''" @click.stop="size='thumbnail'">缩略图</el-button>
                        <el-button :type="size === 'mw690' ? 'primary':''" @click.stop="size='mw690'">中等尺寸</el-button>
                        <el-button :type="size === 'large' ? 'primary':''" @click.stop="size='large'">原图</el-button>
                    </el-button-group>
                </el-col>
                <el-col :span="11" style="text-align:center;">
                    <el-button-group>
                        <el-button :type="clazz === 'link' ? 'primary':''" @click.stop="clazz='link'">图片链接</el-button>
                        <el-button :type="clazz === 'html' ? 'primary':''" @click.stop="clazz='html'">HTML标签</el-button>
                        <el-button :type="clazz === 'ubb' ? 'primary':''" @click.stop="clazz='ubb'">UBB</el-button>
                        <el-button :type="clazz === 'markdown' ? 'primary':''" @click.stop="clazz='markdown'">MarkDown</el-button>
                    </el-button-group> 
                </el-col>
                <el-col :span="5" style="text-align:center;">
                    <el-switch v-model="https" on-text="ON" off-text="OFF"></el-switch> <span class="cursor" @click="https=!https"> HTTPS</span>
                </el-col>
            </el-row>
            <ul id="result">
                <li v-for="(val,key) in uploadedImages" :key="key">
                    <div class="pic" :style="`background: url(${val.src}) center / cover no-repeat;`"></div>
                    <div class="base64">
                        <input readonly @focus="select" :value="val.link">
                    </div>
                </li>
            </ul>
        </div>
        <input id="fileInput" type="file" class="uploadTrigger" style="display:none;" accept="image/*" multiple>
        <el-dialog :close-on-click-modal="false" title="请登录微博" :visible.sync="loginDialogVisible">
            <el-form :model="weibo">
                <el-form-item>
                    <el-input placeholder="邮箱/会员账号/手机号" v-model="weibo.userName" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item>
                    <el-input type="password" placeholder="请输入密码" v-model="weibo.userPwd" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item v-if="pinCodeLink.length">
                    <el-input placeholder="请输入验证码" v-model="weibo.pinCode">
                        <template slot="append"><img @click="rePinCode" width="75" src="http://login.sina.com.cn/cgi/pin.php?r=63185872&s=0&p=12312312"></template>
                    </el-input>
                </el-form-item>
                <el-form-item class="quote"><i class="el-icon-information"></i> 账户信息不会上传，请放心登录</el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer">
                <el-button @click="loginDialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="loginSubmit">登 录</el-button>
            </div>
        </el-dialog>
    </div>
</template>
<script>
import Weibo from '../utils/Weibo'
import Server from '../utils/server'
import qs from 'querystring'
export default{
    data(){
        return {
            size:'large',
            clazz:'link',
            loading:false,
            loadingText:'拼命登录中...',
            loginDialogVisible:false,
            https:false,
            pinCodeLink:'',
            fileLength:0,
            uploadedImages:[],
            storageData:[],
            http_pre:'http://ww',
            https_pre:'https://ws',
            weibo:{
                userName:'',
                userPwd:'',
                pinCode:''
            },
            weiboObj:{}
        }
    },
    mounted(){
        let self = this
        let getZONE = document.querySelectorAll('.drag-wrap')[0]
        let fileInput = document.querySelector('#fileInput')
        let ipc = self.$electron.ipcRenderer
        ipc.on('changeLogin',()=>{
            self.weibo = {
                userName:'',
                userPwd:'',
                pinCode:''
            }
            self.weiboObj = {}
            self.login()
        })
        ipc.on('clearHistory',()=>{
            self.$confirm('确认清除历史记录?','温馨提醒',{
                confirmButtonText:'确认清除',
                cancelButtonText:'取消',
                type: 'warning'
            }).then(()=>{
                localStorage.removeItem('weiboxData')
                self.storageData = []
                self.uploadedImages = []
                self.$message({
                    type: 'success',
                    message: '历史记录清除成功!'
                })
            }).catch(()=>{})
        })
        self.https = localStorage.is_https ?  JSON.parse(localStorage.is_https) : self.https;
        getZONE.addEventListener('dragover', (e) => {
            e.preventDefault()
            getZONE.classList.add('on')
        })
        getZONE.addEventListener('dragleave', (e) => getZONE.classList.remove('on'))
        getZONE.addEventListener('click',(e)=>{
            e.preventDefault()
            fileInput.click()
        })
        getZONE.addEventListener('drop', (e) => {
            e.preventDefault()
            let files = e.dataTransfer.files
            self.putFiles(files)
        })
        fileInput.addEventListener('change',(e)=>{
            e.preventDefault()
            let files = fileInput.files
            self.putFiles(files)
        })
        // 检测登录
        let account = localStorage.weiboAccount ? JSON.parse(localStorage.weiboAccount) : {}
        self.weiboObj = {}
        if(Object.keys(account).length>0){
            self.weibo = account
        }
        let cookies = localStorage.getItem('weiboxCookies') || ''
        if(!cookies){
            self.login()
        }
        // 获取历史
        self.storageData = localStorage.weiboxData ? JSON.parse(localStorage.weiboxData) : []
        self.storageData.forEach(item=>{
            self.uploadedImages.push({
                src:item.src,
                link:self.changeLink(item.src),
                params:item.params
            })
        })
        //HTML5 paste http://www.zhihu.com/question/20893119
        // document.body.addEventListener('paste',(e)=>{
        //     e = e.originalEvent
        //     let clipboardData, items, item
        //     if(e && (clipboardData == e.clipboardData) && (items = clipboardData.items)){
        //         let b = false
        //         for(let i = 0, len = items.length; i < len; i++){
        //             console.log(item)
        //             // if ((item = items[i]) && item.kind == 'file' && item.type.match(/^image\//i)) {
        //             //     b = true;
        //             //     img_file.push(item.getAsFile());
        //             // }
        //         }
        //     }
        // })
    },
    methods:{
        select(event){
            event.target.select()
        },
        login(){
            let self = this
            self.loginDialogVisible = true
        },
        loginSubmit(){
            let self = this
            self.loadingText = '拼命登录中...'
            self.loading = true
            let weibo = Object.keys(self.weiboObj).length == 0 ? new Weibo(self.weibo.userName,self.weibo.userPwd) : self.weiboObj
            if(self.weibo.pinCode){
                weibo.pinCode = self.weibo.pinCode
                self.weiboPostData(weibo)
            }else{
                weibo.getPreLoginData().then(preLoginInitData=>{
                    return weibo.parsePreLoginData(preLoginInitData)
                }).then(preLoginData=>{
                    weibo.preLoginData = preLoginData
                    if(preLoginData['showpin'] == 1){
                        self.pinCodeLink = weibo.getPinImg()
                        self.loading = false
                    }else{
                        self.weiboPostData(weibo)
                    }
                }).catch(ex=>{
                    console.log(ex)
                })
            }
        },
        rePinCode(){
            let self = this
            self.pinCodeLink = self.weiboObj.getPinImg()
        },
        weiboPostData(weibo){
            let self = this
            weibo.postData().then(body=>{
                return weibo.getFinnalLoginUrl(body)
            }).then(finnalRet=>{
                weibo.getCookies(finnalRet.res).then(ret=>{
                    localStorage.weiboxCookies = ret
                    self.loading = false
                    self.loginDialogVisible = false
                    self.$message({
                        message: '微博登录成功~',
                        type: 'success'   
                    })
                    self.saveAccount()
                }).catch(ex=>{
                    console.log(ex)
                })
            }).catch(ex=>{
                console.log(ex)
            })
        },
        saveAccount(){
            let self = this
            self.weibo.pinCode = ''
            localStorage.weiboAccount = JSON.stringify(self.weibo)
        },
        putFiles(files){
            let self = this
            let cookie = localStorage.weiboxCookies || ''
            if(!cookie){
                self.login()
                return
            }
            self.fileLength = 0
            for (let i = 0 ,len = files.length; i < len; i++) {
                let file = files[i]
                if (/image\/\w+/.test(file.type) && file != 'undefined') {
                    ++self.fileLength
                    self.loadingText = '拼命上传中...'
                    self.loading = true
                    self.imageUpload(file)
                }
            }
            if(self.fileLength < 1 && files.length){
                self.$message({
                    message: '很抱歉，不支持非图片的文件格式',
                    type: 'warning'
                })
            }
        },
        imageUpload(file){
            let self = this
            let cookie = localStorage.weiboxCookies || ''
            let reader = new FileReader()
            reader.readAsDataURL(file)
            reader.onloadend = (evt)=>{
                let img = evt.target
                let acceptType = img.result.split(';')[0].toLowerCase()
                let base64 = img.result.split(',')[1]
                let data = new FormData()
                data.append('b64_data',base64)
                let config = {
                    url:'http://picupload.service.weibo.com/interface/pic_upload.php?ori=1&mime=image%2Fjpeg&data=base64&url=0&markpos=1&logo=&nick=0&marks=1&app=miniblog',                    method:'POST',
                    headers:{
                        'Cookie':cookie,
                        'User-Agent':navigator['userAgent']
                    },
                    formData:{
                        b64_data:base64
                    }
                }
                Server(config).then(ret=>{
                    try{
                        let splitIndex,rs,pid,params
                        splitIndex = ret.indexOf('{"')
                        rs = JSON.parse(ret.substring(splitIndex))
                        pid = rs.data.pics.pic_1.pid
                        params = {
                            pid: pid,
                            ext: acceptType == 'data:image/gif' ? '.gif' : '.jpg'
                        }
                        let src = self.pid2url(params)
                        let link = self.changeLink(src)
                        self.save2Local(params,src)
                        self.uploadedImages.unshift({
                            src:src,
                            link:link,
                            params:params
                        })
                        if(--self.fileLength == 0){
                            self.loading = false
                            self.$message({
                                message: '上传成功~',
                                type: 'success'   
                            })
                        }
                    }catch(ex){
                        console.log(ret)
                        console.error(ex)
                        self.loading = false
                        let h = self.$createElement
                        self.$msgbox({
                            title:'温馨提醒',
                            message:h('div',null,[
                                h('p',null,'很抱歉，上传图片失败'),
                                h('p',null,'目测是登录信息失效')
                            ]),
                            showCancelButton: true,
                            confirmButtonText: '重新登录',
                            cancelButtonText: '取消',
                            type: 'warning'
                        }).then(()=>{
                            self.login()
                        }).catch(ex=>{

                        })
                        // self.$message({
                        //     message: '很抱歉，上传失败咯。请查看控制台信息',
                        //     type: 'warning'
                        // })
                        //self.login()
                    }
                })
            }
        },
        pid2url(params) {
            let self = this
            function crc32(str) {
                str = String(str);
                let table = '00000000 77073096 EE0E612C 990951BA 076DC419 706AF48F E963A535 9E6495A3 0EDB8832 79DCB8A4 E0D5E91E 97D2D988 09B64C2B 7EB17CBD E7B82D07 90BF1D91 1DB71064 6AB020F2 F3B97148 84BE41DE 1ADAD47D 6DDDE4EB F4D4B551 83D385C7 136C9856 646BA8C0 FD62F97A 8A65C9EC 14015C4F 63066CD9 FA0F3D63 8D080DF5 3B6E20C8 4C69105E D56041E4 A2677172 3C03E4D1 4B04D447 D20D85FD A50AB56B 35B5A8FA 42B2986C DBBBC9D6 ACBCF940 32D86CE3 45DF5C75 DCD60DCF ABD13D59 26D930AC 51DE003A C8D75180 BFD06116 21B4F4B5 56B3C423 CFBA9599 B8BDA50F 2802B89E 5F058808 C60CD9B2 B10BE924 2F6F7C87 58684C11 C1611DAB B6662D3D 76DC4190 01DB7106 98D220BC EFD5102A 71B18589 06B6B51F 9FBFE4A5 E8B8D433 7807C9A2 0F00F934 9609A88E E10E9818 7F6A0DBB 086D3D2D 91646C97 E6635C01 6B6B51F4 1C6C6162 856530D8 F262004E 6C0695ED 1B01A57B 8208F4C1 F50FC457 65B0D9C6 12B7E950 8BBEB8EA FCB9887C 62DD1DDF 15DA2D49 8CD37CF3 FBD44C65 4DB26158 3AB551CE A3BC0074 D4BB30E2 4ADFA541 3DD895D7 A4D1C46D D3D6F4FB 4369E96A 346ED9FC AD678846 DA60B8D0 44042D73 33031DE5 AA0A4C5F DD0D7CC9 5005713C 270241AA BE0B1010 C90C2086 5768B525 206F85B3 B966D409 CE61E49F 5EDEF90E 29D9C998 B0D09822 C7D7A8B4 59B33D17 2EB40D81 B7BD5C3B C0BA6CAD EDB88320 9ABFB3B6 03B6E20C 74B1D29A EAD54739 9DD277AF 04DB2615 73DC1683 E3630B12 94643B84 0D6D6A3E 7A6A5AA8 E40ECF0B 9309FF9D 0A00AE27 7D079EB1 F00F9344 8708A3D2 1E01F268 6906C2FE F762575D 806567CB 196C3671 6E6B06E7 FED41B76 89D32BE0 10DA7A5A 67DD4ACC F9B9DF6F 8EBEEFF9 17B7BE43 60B08ED5 D6D6A3E8 A1D1937E 38D8C2C4 4FDFF252 D1BB67F1 A6BC5767 3FB506DD 48B2364B D80D2BDA AF0A1B4C 36034AF6 41047A60 DF60EFC3 A867DF55 316E8EEF 4669BE79 CB61B38C BC66831A 256FD2A0 5268E236 CC0C7795 BB0B4703 220216B9 5505262F C5BA3BBE B2BD0B28 2BB45A92 5CB36A04 C2D7FFA7 B5D0CF31 2CD99E8B 5BDEAE1D 9B64C2B0 EC63F226 756AA39C 026D930A 9C0906A9 EB0E363F 72076785 05005713 95BF4A82 E2B87A14 7BB12BAE 0CB61B38 92D28E9B E5D5BE0D 7CDCEFB7 0BDBDF21 86D3D2D4 F1D4E242 68DDB3F8 1FDA836E 81BE16CD F6B9265B 6FB077E1 18B74777 88085AE6 FF0F6A70 66063BCA 11010B5C 8F659EFF F862AE69 616BFFD3 166CCF45 A00AE278 D70DD2EE 4E048354 3903B3C2 A7672661 D06016F7 4969474D 3E6E77DB AED16A4A D9D65ADC 40DF0B66 37D83BF0 A9BCAE53 DEBB9EC5 47B2CF7F 30B5FFE9 BDBDF21C CABAC28A 53B39330 24B4A3A6 BAD03605 CDD70693 54DE5729 23D967BF B3667A2E C4614AB8 5D681B02 2A6F2B94 B40BBE37 C30C8EA1 5A05DF1B 2D02EF8D';
                if (typeof(crc) == 'undefined') {
                    crc = 0;
                }
                let x = 0,
                    y = 0;
                crc = crc ^ (-1);
                for (let i = 0, iTop = str.length; i < iTop; i++) {
                    y = (crc ^ str.charCodeAt(i)) & 0xFF;
                    x = '0x' + table.substr(y * 9, 8);
                    crc = (crc >>> 8) ^ x;
                }
                return crc ^ (-1);
            }

            let url, zone, ext;
            let url_pre = self.https ? self.https_pre : self.http_pre;
            if (typeof(self.size) == 'undefined') self.size = 'bmiddle';
            if (params.pid[9] == 'w') {
                zone = (crc32(params.pid) & 3) + 1;
                url = url_pre + zone + '.sinaimg.cn/' + self.size + '/' + params.pid;
            } else {
                zone = ((params.pid.substr(-2, 2), 16) & 0xf) + 1;
                url = url_pre + zone + '.sinaimg.cn/' + self.size + '/' + params.pid;
            }
            return url + params.ext;
        },
        save2Local(params,src){
            let self = this
            self.storageData.unshift({
                date:Date.now(),
                src:src,
                params:params
            })
            localStorage.weiboxData = JSON.stringify(self.storageData)
        },
        changeLink(src){
            let self = this
            let link = src
            switch(self.clazz){
                case 'html':
                link = `<img src='${src}' />`
                break;
                case 'ubb':
                link = `[img]${src}[/img]`
                break;
                case 'markdown':
                link = `![](${src})`
                break;
            }
            return link
        },
        changePicFormat(){
            let self = this
            let temp = []
            self.uploadedImages.forEach(item=>{
                let src = self.pid2url(item.params)
                let link = self.changeLink(src)
                temp.push({
                    src:src,
                    link:link,
                    params:item.params
                })
            })
            self.uploadedImages = temp
        }
    },
    watch:{
        https(c){
            let self = this
            localStorage.is_https = c
            self.changePicFormat()
        },
        size(c){
            let self = this
            self.changePicFormat()

        },
        clazz(c){
            let self = this
            self.changePicFormat()
        }
    }
}
</script>
<style scoped lang="scss">
.el-dialog{
    .rember{
        color:#b5b4b4;
        cursor: pointer;
    }
    .quote{
        padding-left: 10px;
        border-left: 5px solid #dedede;
        color: #999;
    }
}

*::-webkit-scrollbar{width:0.4rem;background-color:#F5F5F5}
*::-webkit-scrollbar-track{background-color:#F5F5F5;border-radius:3px}
*::-webkit-scrollbar-thumb{border-radius:3px;background-color:#f66}
::selection{background-color:#f66;color:#fff}

.home{
    height:100%;
    font-size:0;
}

.drag-wrap {
    float:left;
    width: 15rem;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    border-right: 1px dashed #ccc;
    font-size:1.2rem;
    position: fixed;
    top:0;
    left:0;
    cursor:pointer;
}
.drag-wrap.on {
    border-color: #f66;
}
.content-wrap{
    overflow: hidden;
    padding:1rem;
    margin-left:15rem;
    position:relative;
    height:100%;
    .options{
        width:100%;
        position:absolute;
        top:0;
        left:0;
        height:5.6rem;
        margin:0 !important;
        box-shadow: 0 1px 15px #dedede;
        background:#fff;
    }
    #result{
        height:100%;
        padding-top:5.6rem;
        overflow:auto;
    }
    .cursor{
        cursor:pointer;
        font-size:1.4rem;
        display:inline-block;
        vertical-align:middle;
        padding-left:.5rem;
    }
    li{
        list-style: none;
        margin: 1rem 0;
        .pic{
            width: 5rem;
            height: 5rem;
            float: left;
            border: 1px dashed #ccc;
            border-radius: 3px;
            margin-right: 1rem;
        }
        .base64{
            height: 5rem;
            overflow: hidden;
            border: 1px dashed #ccc;
            border-radius: 3px;
            padding: 0.5rem;
            input{
                width: 100%;
                height: 100%;
                overflow-y: auto;
                border: 0;
                resize: none;
                outline: none;
                font-size:1.5rem;
            }
        }
    }
}
</style>