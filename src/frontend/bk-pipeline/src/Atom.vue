<template>
    <li
        :key="atom.id"
        :class="atomCls"
        @click.stop="handleAtomClick"
    >
        <template
            v-if="isQualityGateAtom"
        >
            <span class="atom-title">
                <i></i>
                <span>{{ t('quality') }}</span>
                <i></i>
            </span>
            <template v-if="atom.isReviewing">
                <Logo v-if="isBusy" name="circle-2-1" size="14" class="spin-icon" />
                <span
                    v-else
                    :class="{
                        'handler-list': true,
                        'disabled-review': atom.isReviewing && !hasReviewPerm
                    }"
                >
                    <span class="revire-btn" @click.stop="qualityApprove('PROCESS')">{{ t('resume') }}</span>
                    <span class="review-btn" @click.stop="qualityApprove('ABORT')">{{ t('terminate') }}</span>
                </span>
            </template>
        </template>
        <template
            v-else
        >
            <status-icon
                v-if="!isSkip && !!atomStatus"
                type="element"
                :status="atomStatus"
                :is-hook="isHookAtom"
            />
            
            <img
                v-else-if="atom.logoUrl"
                :src="atom.logoUrl"
                :class="logoCls"
            />
            <Logo v-else
                :class="logoCls"
                :name="svgAtomIcon"
                size="18"
            />
            <p class="atom-name">
                <span
                    :title="atom.name"
                    :class="skipSpanCls"
                >
                    {{ atom.atomCode ? atom.name : t('pendingAtom') }}
                </span>
            </p>
            <span v-if="isExecuting">{{ execTime }}</span>
            <Logo v-if="isBusy" name="circle-2-1" size="14" class="spin-icon" />
            <bk-popover v-else-if="atom.isReviewing" placement="top">
                <span
                    @click.stop="reviewAtom"
                    class="atom-reviewing-tips atom-operate-area"
                    :disabled="!hasReviewPerm"
                >
                    {{ t('manualCheck') }}
                </span>
                <template slot="content">
                    <p>
                        {{ t('checkUser') }}{{ atom.computedReviewers.join(';') }}
                    </p>
                </template>
            </bk-popover>
            <bk-popover v-else-if="isReviewAbort" placement="top">
                <span class="atom-review-diasbled-tips">{{ t('aborted') }}</span>
                <template slot="content">
                    <p>
                        {{ t('abortTips') }}{{ t('checkUser') }}{{ cancelUserId }}
                    </p>
                </template>
            </bk-popover>
            <template v-else-if="atom.status === 'PAUSE'">
                <bk-popover placement="top" :disabled="!Array.isArray(atom.pauseReviewers)">
                    <span
                        :class="resumeSpanCls"
                        @click.stop="atomExecute(true)"
                    >
                        {{ t('resume') }}
                    </span>
                    <template slot="content">
                        <p>{{ t('checkUser') }}{{ pauseReviewerStr }}</p>
                    </template>
                </bk-popover>
                <span
                    @click.stop="atomExecute(false)"
                    class="pause-button"
                >
                    <span>{{ t('stop') }}</span>
                </span>
            </template>
            <span class="atom-operate-area">
                <span
                    v-if="atom.canRetry && !isBusy"
                    @click.stop="skipOrRetry(false)"
                >
                    {{ t('retry') }}
                </span>
                <span
                    v-if="atom.canSkip && !isBusy"
                    @click.stop="skipOrRetry(true)"
                >
                    {{ t('SKIP') }}
                </span>
                <bk-popover
                    v-if="!isSkip && !isWaiting && atom.elapsed && !atom.canSkip && !atom.canRetry"
                    placement="top"
                    :disabled="!atom.elapsed"
                >
                    <span class="atom-execute-time">
                        <span v-if="isElapsedGt1h">&gt;</span>
                        {{ isElapsedGt1h ? '1h' : formatTime }}
                    </span>
                    <template slot="content">
                        <p>{{ formatTime }}</p>
                    </template>
                </bk-popover>
            </span>
            
            <Logo
                v-if="editable && stageIndex !== 0 && !atom.isError"
                name="clipboard"
                class="copy"
                size="14"
                :title="t('copyAtom')"
                @click.stop="copyAtom"
            />
            
            <template v-if="editable">
                <i @click.stop="deleteAtom(false)" class="add-plus-icon close" />
                <Logo v-if="atom.isError" class="atom-invalid-icon" name="exclamation-triangle-shape" />
            </template>
            <span v-if="canSkipElement" @click.stop="">
                <bk-checkbox class="atom-canskip-checkbox" v-model="atom.canElementSkip" :disabled="isSkip" />
            </span>
        </template>
    </li>
</template>

<script>
    import { bkPopover, bkCheckbox } from 'bk-magic-vue'
    import StatusIcon from './StatusIcon'
    import Logo from './Logo'
    import {
        eventBus,
        hashID,
        randomString,
        isTriggerContainer,
        convertMStoString
    } from './util'
    import { localeMixins } from './locale'
    import {
        CLICK_EVENT_NAME,
        COPY_EVENT_NAME,
        DELETE_EVENT_NAME,
        QUALITY_IN_ATOM_CODE,
        QUALITY_OUT_ATOM_CODE,
        ATOM_CONTINUE_EVENT_NAME,
        ATOM_EXEC_EVENT_NAME,
        ATOM_QUALITY_CHECK_EVENT_NAME,
        ATOM_REVIEW_EVENT_NAME,
        STATUS_MAP
    } from './constants'

    export default {
        name: 'atom',
        components: {
            StatusIcon,
            Logo,
            bkPopover,
            bkCheckbox
        },
        mixins: [localeMixins],
        props: {
            stage: {
                type: Object,
                requiured: true
            },
            container: {
                type: Object,
                requiured: true
            },
            atom: {
                type: Object,
                requiured: true
            },
            stageIndex: {
                type: Number,
                requiured: true
            },
            containerIndex: {
                type: Number,
                requiured: true
            },
            containerGroupIndex: Number,
            atomIndex: {
                type: Number,
                requiured: true
            },
            isWaiting: Boolean,
            editable: Boolean,
            containerDisabled: Boolean,
            isLastAtom: Boolean,
            prevAtom: {
                type: Object
            },
            cancelUserId: {
                type: String,
                default: 'unknow'
            },
            userName: {
                type: String,
                default: 'unknow'
            },
            canSkipElement: Boolean,
            matchRules: {
                type: Array,
                default: () => []
            }
        },
        data () {
            return {
                isBusy: false,
                timer: null,
                execTime: this.atom.startEpoch ? convertMStoString(Date.now() - this.atom.startEpoch) : '--'
            }
        },
        computed: {
            skipSpanCls () {
                return { 'skip-name': this.isSkip }
            },
            resumeSpanCls () {
                return {
                    disabled: this.isBusy || !this.hasExecPerm,
                    'pause-button': true
                }
            },
            isReviewAbort () {
                return this.atom.status === STATUS_MAP.REVIEW_ABORT
            },
            isHookAtom () {
                try {
                    return !!this.atom.additionalOptions.elementPostInfo
                } catch (error) {
                    return false
                }
            },
            isSkip () {
                try {
                    return this.atom.status === 'SKIP' || !this.atom.additionalOptions.enable || this.containerDisabled
                } catch (error) {
                    return false
                }
            },
            qualityStatus () {
                switch (true) {
                    case [STATUS_MAP.SUCCEED, STATUS_MAP.REVIEW_PROCESSED].includes(this.atom.status):
                        return STATUS_MAP.SUCCEED
                    case [STATUS_MAP.QUALITY_CHECK_FAIL, STATUS_MAP.REVIEW_ABORT].includes(this.atom.status):
                        return STATUS_MAP.FAILED
                }
                return ''
            },
            isQualityGateAtom () {
                return this.isQualityGate(this.atom)
            },
            isLastQualityAtom () {
                return this.atom.atomCode === QUALITY_OUT_ATOM_CODE && this.isLastAtom
            },
            isPrevAtomQuality () {
                return this.prevAtom !== null && this.isQualityGate(this.prevAtom)
            },
            
            atomStatus () {
                try {
                    if (this.atom.status) {
                        return this.atom.status
                    }
                    return this.isWaiting ? STATUS_MAP.WAITING : ''
                } catch (error) {
                    return ''
                }
            },
            atomStatusCls () {
                try {
                    if (this.atom.additionalOptions && this.atom.additionalOptions.enable === false) {
                        return STATUS_MAP.DISABLED
                    }
                    return this.atomStatus
                } catch (error) {
                    console.error('get atom cls error', error)
                    return ''
                }
            },
            logoCls () {
                return {
                    'atom-icon': true,
                    'skip-icon': this.isSkip
                }
            },
            atomCls () {
                return {
                    readonly: !this.editable,
                    'bk-pipeline-atom': true,
                    'trigger-atom': isTriggerContainer(this.container),
                    [STATUS_MAP.REVIEWING]: this.atom.isReviewing,
                    [this.qualityStatus]: this.isQualityGateAtom && !!this.qualityStatus,
                    [this.atomStatusCls]: !!this.atomStatusCls,
                    'quality-atom': this.isQualityGateAtom,
                    'is-error': this.atom.isError,
                    'is-intercept': this.isQualityCheckAtom,
                    'template-compare-atom': this.atom.templateModify,
                    'last-quality-atom': this.isLastQualityAtom,
                    'quality-prev-atom': this.isPrevAtomQuality
                }
            },
            svgAtomIcon () {
                if (this.isHookAtom) {
                    return 'build-hooks'
                }
                const { atomCode } = this.atom
                if (!atomCode) {
                    return 'placeholder'
                }
                return atomCode
            },
            hasReviewPerm () {
                return this.atom.computedReviewers.includes(this.userName)
            },
            hasExecPerm () {
                const hasPauseReviewer = Array.isArray(this.atom.pauseReviewers)
                if (!hasPauseReviewer || (hasPauseReviewer && this.atom.pauseReviewers.length === 0)) return true
                return this.atom.pauseReviewers.includes(this.userName)
            },
            pauseReviewerStr () {
                return Array.isArray(this.atom.pauseReviewers) && this.atom.pauseReviewers.join(';')
            },
            isElapsedGt1h () {
                return this.atom.elapsed && this.atom.elapsed >= 36e5
            },
            formatTime () {
                try {
                    return convertMStoString(this.atom.elapsed)
                } catch (error) {
                    return '--'
                }
            },
            isQualityCheckAtom () {
                return Array.isArray(this.matchRules)
                    && this.matchRules.some(rule => rule.taskId === this.atom.atomCode
                        && (rule.ruleList.some(val => this.atom.name.indexOf(val.gatewayId) > -1) || rule.ruleList.every(val => !val.gatewayId))
                    )
            },
            isExecuting () {
                return this.atomStatus === STATUS_MAP.RUNNING && this.atom.startEpoch
            }
        },
        watch: {
            isExecuting (v) {
                if (v) {
                    this.executeCounter()
                } else {
                    clearInterval(this.timer)
                }
            }
        },
        mounted () {
            if (this.isExecuting) {
                this.executeCounter()
            }
        },
        beforeDestroy () {
            clearInterval(this.timer)
        },
        methods: {
            executeCounter () {
                clearInterval(this.timer)
                this.timer = setInterval(() => {
                    this.execTime = convertMStoString(Date.now() - this.atom.startEpoch)
                }, 1000)
            },
            reviewAtom () {
                eventBus.$emit(ATOM_REVIEW_EVENT_NAME, this.atom)
            },
            isQualityGate (atom) {
                try {
                    return [QUALITY_IN_ATOM_CODE, QUALITY_OUT_ATOM_CODE].includes(atom.atomCode)
                } catch (error) {
                    return false
                }
            },
            handleAtomClick () {
                eventBus.$emit(CLICK_EVENT_NAME, {
                    stageIndex: this.stageIndex,
                    containerIndex: this.containerIndex,
                    containerGroupIndex: this.containerGroupIndex,
                    elementIndex: this.atomIndex
                })
            },
            copyAtom () {
                const { id, stepId, ...restAttr } = this.atom
                this.$emit(COPY_EVENT_NAME, {
                    elementIndex: this.atomIndex,
                    element: JSON.parse(JSON.stringify({
                        ...restAttr,
                        stepId: randomString(3),
                        id: `e-${hashID()}`
                    }))
                })
            },
            deleteAtom () {
                this.$emit(DELETE_EVENT_NAME, {
                    elementIndex: this.atomIndex
                })
            },
            async atomExecute (isContinue = false) {
                if (this.isBusy || !this.hasExecPerm) return
        
                this.isBusy = true
                const { stageIndex, containerIndex, containerGroupIndex, atomIndex } = this

                await this.asyncEvent(ATOM_EXEC_EVENT_NAME, {
                    stageIndex,
                    containerIndex,
                    containerGroupIndex,
                    isContinue,
                    showPanelType: 'PAUSE',
                    elementIndex: atomIndex,
                    stageId: this.stage.id,
                    containerId: this.container.id,
                    taskId: this.atom.id,
                    atom: this.atom
                })
                setTimeout(() => {
                    this.isBusy = false
                }, 1000)
            },
            
            async qualityApprove (action) {
                if (this.hasReviewPerm) {
                    try {
                        this.isBusy = true
                        const data = {
                            elementId: this.atom.id,
                            action
                        }
                        await this.asyncEvent(ATOM_QUALITY_CHECK_EVENT_NAME, data)
                    } catch (error) {
                        console.error(error)
                    } finally {
                        setTimeout(() => {
                            this.isBusy = false
                        }, 1000)
                    }
                }
            },

            asyncEvent (...args) {
                return new Promise((resolve, reject) => {
                    eventBus.$emit(...args, resolve, reject)
                })
            },
            
            async skipOrRetry (skip = false) {
                if (this.isBusy) return
                try {
                    this.isBusy = true
                        
                    await this.asyncEvent(ATOM_CONTINUE_EVENT_NAME, {
                        taskId: this.atom.id,
                        skip
                    })
                } catch (error) {
                    console.error(error)
                } finally {
                    setTimeout(() => {
                        this.isBusy = false
                    }, 1000)
                }
            }
            
        }
    }
</script>

<style lang="scss">
    @import "./conf";
    .bk-pipeline .bk-pipeline-atom {
    cursor: pointer;
    position: relative;
    display: flex;
    flex-direction: row;
    align-items: center;
    height: $itemHeight;
    margin: 0 0 11px 0;
    background-color: white;
    border-radius: 2px;
    font-size: 14px;
    transition: all .4s ease-in-out;
    z-index: 2;
    border: 1px solid $fontLighterColor;

    &.trigger-atom {
        &:before,
        &:after {
            display: none;
        }
    }

    &:first-child {
        &:before {
            top: -16px;
        }
    }
    &:before {
        content: '';
        position: absolute;
        height: 14px;
        width: 2px;
        background: $fontLighterColor;
        top: -12px;
        left: 22px;
        z-index: 1;
    }

    &:after {
        content: '';
        position: absolute;
        height: 4px;
        width: 4px;
        border: 2px solid $fontLighterColor;
        border-radius: 50%;
        background: white !important;
        top: -5px;
        left: 19px;
        z-index: 2;
    }

    &.is-intercept {
        border-color: $warningColor;
        &:hover {
            border-color: $warningColor;
        }
    }

    &.is-error {
        border-color: $dangerColor;
        color: $dangerColor;
        &:hover {
            .atom-invalid-icon {
                display: none;
            }
        }
        .atom-invalid-icon {
            margin: 0 12px;
        }
    }

    &:not(.readonly):hover{
        border-color: $primaryColor;
        .atom-icon.skip-icon {
            color: $fontLighterColor;
        }
        .atom-icon,
        .atom-name {
            color: $primaryColor;
        }
        .add-plus-icon.close,
        .copy {
            cursor: pointer;
            display: block;
        }
    }
    .atom-icon {
        text-align: center;
        margin: 0 14.5px;
        font-size: 18px;
        width: 18px;
        color: $fontWeightColor;
        fill: currentColor;
    }
    .atom-icon.skip-icon {
        color: $fontLighterColor;
    }
    .atom-name span.skip-name {
        text-decoration: line-through;
        color: $fontLighterColor;
        &:hover {
            color: $fontLighterColor;
        }
    }
    .pause-button {
        margin-right: 8px;
        color: $primaryColor;
    }

    .add-plus-icon.close {
        @include add-plus-icon(#fff, #fff, #c4c6cd, 16px, true);
        @include add-plus-icon-hover($dangerColor, $dangerColor, white);
        display: none;
        margin-right: 10px;
        border: none;
        transform: rotate(45deg);
        &:before, &:after {
            left: 7px;
            top: 4px;
        }
    }

    .copy {
        display: none;
        margin-right: 10px;
        color: $fontLighterColor;
        &:hover {
            color: $primaryColor;
        }
    }

    > .atom-name {
        flex: 1;
        color: $fontWeightColor;
        @include ellipsis();
        max-width: 188px;
        span:hover {
            color: $primaryColor;
        }
    }
    .disabled {
        cursor: not-allowed;
        color: $fontLighterColor;
    }

    .atom-operate-area {
        margin: 0 8px 0 2px;
        color: $primaryColor;
        font-size: 12px;
    }

    .atom-reviewing-tips {
        &[disabled] {
            cursor: not-allowed;
            color: #c3cdd7;
        }
    }

    .atom-review-diasbled-tips {
        color: #c3cdd7;
        margin: 0 8px 0 2px;
    }
    
    .atom-canskip-checkbox {
        margin-right: 6px;
    }
    
    &.quality-atom {
        display: flex;
        justify-content: center;
        border-color: transparent;
        height: 24px;
        background: transparent;
        border-color: transparent !important;
        font-size: 12px;
        &:before {
            height: 40px;
            z-index: 8;
        }
        &:after {
            display: none;
        }
        &.last-quality-atom {
           &:before {
                height: 22px;
            }
        }
        .atom-title {
            display: flex;
            width: 100%;
            align-items: center;
            justify-content: center;
            margin-left: 22px;
            > span {
                border-radius: 12px;
                font-weight: bold;
                border: 1px solid $fontLighterColor;
                padding: 0 12px;
                margin: 0 4px;
            }
            > i {
                height: 0;
                flex: 1;
                border-top: 2px dashed $fontLighterColor;
            }
        }
        
        .handler-list {
            position: absolute;
            right: 0;
            span {
                color: $primaryColor;
                font-size: 12px;
                &:first-child {
                    margin-right: 5px;
                }
            }
        }
        .executing-job {
            position: absolute;
            top: 6px;
            right: 42px;
            &:before {
                display: inline-block;
                animation: rotating infinite .6s ease-in-out;
            }
        }
        .disabled-review span {
            color: $fontLighterColor;
            cursor: default;
        }
    }
    &.readonly {
        background-color: white;
        .atom-name:hover {
            span {
                color: $fontWeightColor;
            }
            .skip-name {
                text-decoration: line-through;
                color: $fontLighterColor;
            }
        }
        
        &.quality-prev-atom {
            &:before {
                height: 24px;
                top: -23px;
            }
        }
        
    }
}

</style>
