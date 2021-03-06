<template>
    <div class="form-wrap">
        <Form :model="formItem" :label-width="100">
            <FormItem label="APK文件:">
                <Upload
                        type="drag"
                        paste
                        :before-upload="onBeforeUpload"
                        :on-success="onUploadSuccess"
                        :on-error="onUploadError"
                        :action="formItem.baseUrl+'/file/uploadImages.json'"
                        :headers="header"
                >
                    <div style="padding: 20px 0">
                        <Icon type="ios-cloud-upload" size="52" style="color: #3399ff"></Icon>
                        <p>点击选择文件或者拖拽文件上传</p>
                    </div>
                </Upload>
            </FormItem>

            <!--            <FormItem v-if="formItem.isUploadFile">-->
            <!--                <Spin>-->
            <!--                    <Icon type="ios-loading" size=18 class="demo-spin-icon-load"></Icon>-->
            <!--                    <div>正在上传...</div>-->
            <!--                </Spin>-->
            <!--            </FormItem>-->

            <FormItem v-if="apkInfo">
                <Avatar shape="square" icon="ios-person" size="large" :src="apkInfo.icon"/>
            </FormItem>

            <FormItem v-if="apkInfo">
                <p>{{apkInfo.application.label[0]}}</p>
            </FormItem>

            <FormItem v-if="apkInfo" label="版本号:">
                <p class="apk_info">{{apkInfo.versionCode}}</p>
            </FormItem>

            <FormItem v-if="apkInfo" label="版本名:">
                <p class="apk_info">{{apkInfo.versionName}}</p>
            </FormItem>

            <FormItem label="APK Url地址:">
                <Input v-model="formItem.apkUrl" placeholder="请输入url地址(拖拽自动上传)"/>
            </FormItem>

            <FormItem label="服务器地址:">
                <Input v-model="formItem.baseUrl" placeholder="请输入服务器地址"/>
            </FormItem>

            <FormItem label="可选服务器:" class="apk_info">
                <Select @on-change="onUrlChange">
                    <Option v-for="item in urls" :value="item.url" :key="item.url">{{ item.label +' '+ item.url}}
                    </Option>
                </Select>
            </FormItem>

            <FormItem label="登录接口:">
                <Input v-model="formItem.loginUrl" placeholder="请输入接口地址"/>
            </FormItem>

            <FormItem label="版本更新接口:">
                <Input v-model="formItem.apiUrl" placeholder="请输入接口地址"/>
            </FormItem>

            <FormItem label="更新类型:" class="apk_info">
                <Select v-model="coerce" style="width:200px">
                    <Option v-for="item in coerceList" :value="item.value" :key="item.value">{{ item.label }}</Option>
                </Select>
            </FormItem>

            <FormItem label="更新说明:">
                <Input v-model="description"
                       type="textarea"
                       :rows="4"
                       placeholder="请输入版本更新说明:"/>
            </FormItem>

            <FormItem label="提交数据预览:">
                <Input v-model="postBody"
                       type="textarea"
                       :autosize="{minRows: 6,maxRows: 50}"
                       placeholder="最终提交的数据"/>
            </FormItem>

            <FormItem>
                <Button type="primary" @click="updateVersion" style="width: 250px; height: 40px">提交</Button>
            </FormItem>
        </Form>
    </div>
</template>

<script>
    import axios from "axios"
    import login from "../login"
    import ApkParser from 'app-info-parser/src/apk'
    import urls from '../../public/json/urls'

    export default {
        data() {
            return {
                header: {
                    'Authorization': localStorage.getItem("token")
                },
                formItem: {
                    //选择的文件
                    file: null,
                    //是否正在上传文件
                    isUploadFile: false,
                    baseUrl: "",
                    apkUrl: "",
                    apiUrl: "/sys-app-version/add.json",
                    loginUrl: "/oauth/token",
                },
                //更新描述
                description: "1. 修复已知问题",
                apkInfo: null,
                //0:非强制更新 1:强制更新 -1:内测用户更新
                coerce: 0,
                coerceList: [{label: "非强制更新", value: 0}, {label: "强制更新", value: 1}, {label: "内测用户更新", value: -1}],
                urls: urls
            };
        },
        mounted() {
            this.formItem.baseUrl = urls[0].url
            document.title = "版本发布"
        },
        computed: {
            postBody: function () {
                if (this.apkInfo) {
                    let versionName = this.apkInfo.versionName
                    if (this.coerce === -1) {
                        versionName += "(内测版)"
                    }
                    return JSON.stringify({
                        code: this.apkInfo.versionCode,
                        name: versionName,
                        description: this.description,
                        downloadUrl: this.formItem.apkUrl,
                        coerce: this.coerce
                    }, null, 2)
                } else {
                    return JSON.stringify({
                        description: this.description,
                        downloadUrl: this.formItem.apkUrl,
                        coerce: this.coerce
                    }, null, 2)
                }
            }
        },
        watch: {
            'formItem.loginUrl': function () {
                login(this.formItem.loginUrl)
            },
            'formItem.baseUrl': function () {
                login(this.formItem.loginUrl)
            }
        },
        methods: {
            onUrlChange(value) {
                localStorage.setItem("baseUrl", value)
                localStorage.setItem("token", '')
                this.formItem.baseUrl = value
            },
            onBeforeUpload(file) {
                this.formItem.file = file
                this.formItem.isUploadFile = true

                this.header['Authorization'] = localStorage.getItem("token")

                const parser = new ApkParser(file)
                parser.parse().then(result => {
                    // eslint-disable-next-line no-console
                    console.log('app info ----> ', result)
                    this.apkInfo = result
                }).catch(err => {
                    // eslint-disable-next-line no-console
                    console.log('err ----> ', err)
                })
                return true
            },
            onUploadSuccess(response) {
                this.formItem.isUploadFile = false
                this.formItem.apkUrl = this.formItem.baseUrl + response.data.filePath
            },
            onUploadError(error) {
                this.formItem.isUploadFile = false
            },
            updateVersion() {
                this.header['Authorization'] = localStorage.getItem("token")

                if (!this.header['Authorization']) {
                    this.$Message.error("登录失败, 请刷新重试..")
                    return
                }

                if (!this.apkInfo) {
                    this.$Message.error("请先上传APK文件.")
                    return
                }

                if (this.formItem.isUploadFile) {
                    this.$Message.error("请等待文件上传完成.")
                    return
                }

                this.header["Content-Type"] = "application/json;charset=UTF-8"
                let self = this

                let baseUrl = localStorage.getItem("baseUrl")

                axios.post(this.formItem.apiUrl, this.postBody, {
                    headers: this.header,
                    baseURL: baseUrl,
                }).then(function (response) {
                    if (response.data.code < 300 && response.data.code >= 200) {
                        self.$Message.success("版本发布完成!.")
                    } else {
                        self.$Message.error(response.data.msg)
                    }
                }).catch(function (error) {
                    self.$Message.error(error)
                });
            }
        }
    };
</script>

<style>
    .form-wrap {
        margin: 0 auto;
        width: 800px;
    }

    .ivu-form-item {
        margin-bottom: 10px;
    }

    /*.ivu-upload {*/
    /*    float: left;*/
    /*}*/

    .demo-spin-icon-load {
        animation: ani-demo-spin 1s linear infinite;
    }

    .apk_info {
        text-align: left;
    }

    @keyframes ani-demo-spin {
        from {
            transform: rotate(0deg);
        }
        50% {
            transform: rotate(180deg);
        }
        to {
            transform: rotate(360deg);
        }
    }
</style>