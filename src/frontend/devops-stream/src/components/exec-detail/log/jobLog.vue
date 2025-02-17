<template>
    <section class="job-log">
        <bk-log-search :execute-count="executeCount" @change-execute="changeExecute" class="log-tools">
            <template #tool>
                <li class="more-button" @click="toggleShowDebugLog">{{ showDebug ? $t('pipeline.showLog') : $t('pipeline.hideLog')}}</li>
                <li class="more-button" @click="downloadLog">{{$t('pipeline.downloadLog')}}</li>
            </template>
        </bk-log-search>
        <bk-multiple-log ref="multipleLog"
            class="bk-log"
            :log-list="pluginList"
            @open-log="openLog"
            @tag-change="tagChange"
        >
            <template slot-scope="log">
                <status-icon :status="log.data.status" :is-hook="((log.data.additionalOptions || {}).elementPostInfo || false)" class="multiple-log-status"></status-icon>
                {{ log.data.name }}
            </template>
        </bk-multiple-log>
    </section>
</template>

<script>
    import statusIcon from '../status'
    import { uuid } from '@/utils'
    import { pipelines } from '@/http'
    import { mapState } from 'vuex'

    export default {
        components: {
            statusIcon
        },

        props: {
            buildId: {
                type: String
            },
            pluginList: {
                type: Array
            },
            downLoadLink: {
                type: String
            },
            executeCount: {
                type: Number
            }
        },

        data () {
            return {
                logPostData: {},
                closeIds: [],
                curExe: this.executeCount,
                showDebug: false,
                hasRetryGetLog: false
            }
        },

        computed: {
            ...mapState(['projectId'])
        },

        beforeDestroy () {
            this.closeLog()
        },

        methods: {
            toggleShowDebugLog () {
                this.showDebug = !this.showDebug
                this.clearAllLog()
                this.$refs.multipleLog.foldAllPlugin()
            },

            changeExecute (execute) {
                this.curExe = execute
                this.clearAllLog()
            },

            clearAllLog () {
                this.closeLog()
                const ref = this.$refs.multipleLog
                Object.keys(this.logPostData).forEach((id) => {
                    ref.changeExecute(id)
                })
                this.logPostData = {}
            },

            tagChange (tag, id) {
                const ref = this.$refs.multipleLog
                const postData = this.logPostData[id]
                clearTimeout(postData.timeId)
                this.closeIds.push(postData.hashId)
                ref.changeExecute(id)
                postData.lineNo = 0
                postData.subTag = tag
                this.getLog(id, postData)
            },

            closeLog () {
                Object.keys(this.logPostData).forEach(key => {
                    const postData = this.logPostData[key]
                    this.closeIds.push(postData.hashId)
                    clearTimeout(postData.timeId)
                })
            },

            openLog (plugin) {
                const id = plugin.id
                let postData = this.logPostData[id]
                if (!postData) {
                    postData = this.logPostData[id] = {
                        projectId: this.projectId,
                        pipelineId: this.$route.params.pipelineId,
                        buildId: this.buildId,
                        tag: id,
                        currentExe: this.curExe,
                        lineNo: 0,
                        debug: this.showDebug
                    }

                    this.$nextTick(() => {
                        this.getLog(id, postData)
                    })
                }
            },

            getLog (id, postData) {
                const hashId = postData.hashId = uuid()
                let logMethod = pipelines.getAfterLog
                if (postData.lineNo <= 0) logMethod = pipelines.getInitLog
                const ref = this.$refs.multipleLog

                logMethod(postData).then((res) => {
                    if (this.closeIds.includes(hashId)) return
                    if (res.status !== 0) {
                        let errMessage
                        switch (res.status) {
                            case 1:
                                errMessage = this.$t('pipeline.logEmpty')
                                break
                            case 2:
                                errMessage = this.$t('pipeline.logCleared')
                                break
                            case 3:
                                errMessage = this.$t('pipeline.logClosed')
                                break
                            default:
                                errMessage = this.$t('pipeline.logErr')
                        }
                        ref.handleApiErr(errMessage, id)
                        return
                    }

                    const subTags = res.subTags
                    if (subTags && subTags.length > 0) {
                        const tags = subTags.map((tag) => ({ label: tag, value: tag }))
                        tags.unshift({ label: 'All', value: '' })
                        ref.setSubTag(tags, id)
                    }

                    const logs = res.logs || []
                    const lastLog = logs[logs.length - 1] || {}
                    const lastLogNo = lastLog.lineNo || postData.lineNo - 1 || -1
                    postData.lineNo = +lastLogNo + 1

                    if (res.finished) {
                        if (res.hasMore) {
                            ref.addLogData(logs, id)
                            postData.timeId = setTimeout(() => this.getLog(id, postData), 100)
                        } else {
                            ref.addLogData(logs, id)
                            if (!this.hasRetryGetLog) {
                                this.hasRetryGetLog = true
                                postData.timeId = setTimeout(() => this.getLog(id, postData), 3000)
                            }
                        }
                    } else {
                        ref.addLogData(logs, id)
                        postData.timeId = setTimeout(() => this.getLog(id, postData), 1000)
                    }
                }).catch((err) => {
                    this.$bkMessage({ theme: 'error', message: err.message || err })
                    if (ref) ref.handleApiErr(err.message, id)
                })
            },

            downloadLog () {
                const downloadLink = `${this.downLoadLink}&executeCount=${this.curExe}`
                location.href = downloadLink
            }
        }
    }
</script>

<style lang="postcss" scoped>
    .job-log {
        height: calc(100% - 59px);
    }

    .log-tools {
        position: absolute;
        right: 20px;
        top: 13px;
        display: flex;
        align-items: center;
        line-height: 30px;
        user-select: none;
        background: none;
    }

    .multiple-log-status {
        width: 14px;
        height: 15px;
        margin: 0 9px;
        padding: 1px 0;
        /deep/ svg {
            width: 14px;
            height: 14px;
        }
    }
</style>
