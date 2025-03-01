<template>
    <div class="modal">
        <div class="modal-mask">
            <div class="modal-wrapper">
                <div class="modal-container" @keydown.27="cancel()" @keydown.enter="save()">
                    <a class="inline close-button" href="javascript:void(0);" @click="cancel()"><i class="fas fa-times"/></a>
                    <a class="close-button" href="javascript:;" @click="openHelp()"><i class="fas fa-question-circle"></i></a>
                    <div class="modal-header">
                        <h1 name="header">{{ $t('sequentialInput') }}</h1>
                    </div>

                    <div class="modal-body" v-if="inputConfig">
                        <div class="srow">
                            <span>{{ $t('sequentialInputMethod2InputEvents') }}</span>
                            <a :aria-label="$t('help')" href="javascript:;" @click="openHelp()"><i class="fas blue fa-question-circle"></i></a>
                        </div>
                        <div class="srow" >
                            <div class="twelve columns">
                                <input v-focus type="checkbox" id="enableSeqinput" v-model="inputConfig.seqEnabled"/>
                                <label class="inline" for="enableSeqinput">{{ $t('enableSequentialInput') }}</label>
                            </div>
                        </div>
                        <div v-show="inputConfig.seqEnabled">
                            <accordion :acc-label="$t('input')" acc-open="true" acc-label-type="h2" acc-background-color="white" class="srow">
                                <input-event-list v-model="inputConfig.seqInputs" :input-labels="[InputConfig.NEXT_ELEMENT, InputConfig.PREVIOUS_ELEMENT, InputConfig.SELECT]" :error-inputs="errorInputs" @input="inputChanged"></input-event-list>
                                <div class="srow">
                                    <button class="twelve columns" @click="resetInput">{{ $t('resetToDefaultInputConfiguration') }}</button>
                                </div>
                            </accordion>
                            <accordion :acc-label="$t('ADVANCED_SETTINGS')" acc-label-type="h2" acc-background-color="white">
                                <div class="srow">
                                    <div class="twelve columns">
                                        <input type="checkbox" id="chkResetToStart" v-model="inputConfig.seqResetToStart"/>
                                        <label for="chkResetToStart">{{ $t('goToStartPositionAfterSelect') }}</label>
                                    </div>
                                </div>
                                <div class="srow">
                                    <div class="twelve columns">
                                        <input type="checkbox" id="chkAutoScanning" v-model="inputConfig.seqAuto"/>
                                        <label for="chkAutoScanning">{{ $t('automaticTimedSequentialInput') }}</label>
                                    </div>
                                </div>
                                <div class="srow" v-show="inputConfig.seqAuto">
                                    <label class="four columns" for="inScanTime">{{ $t('scanningTimeMs') }}</label>
                                    <input type="range" id="inScanTime" v-model.number="inputConfig.seqTimeoutMs" min="100" max="6000" step="100"/>
                                    <input type="number" v-model.number="inputConfig.seqTimeoutMs" min="100" max="6000" step="100"/>
                                </div>
                                <div class="srow" v-show="inputConfig.seqAuto">
                                    <label class="four columns" for="inFirstElement">{{ $t('timeFactorFirstElement') }}</label>
                                    <input type="range" id="inFirstElement" v-model.number="inputConfig.seqTimeoutFirstElementFactor" min="1" max="5" step="0.1"/>
                                    <input type="number" v-model.number="inputConfig.seqTimeoutFirstElementFactor" min="1" max="5" step="0.5" />
                                </div>
                            </accordion>
                            <accordion :acc-label="$t('generalInputSettings')" acc-label-type="h2" acc-background-color="white">
                                <global-input-options :input-config="inputConfig"/>
                            </accordion>
                            <accordion :acc-label="$t('TEST_CONFIGURATION')" acc-label-type="h2" acc-background-color="white" @open="testOpen = true; initTest()" @close="testOpen = false; stopTest()">
                                <test-area :selected-element="selectedTestElement"></test-area>
                            </accordion>

                            <div class="warn" v-show="error">
                                <i class="fas fa-exclamation-triangle"></i> {{error}}
                            </div>
                        </div>
                    </div>

                    <div class="modal-footer">
                        <div class="button-container row">
                            <button @click="cancel()" class="four columns offset-by-four">
                                <i class="fas fa-times"/> <span>{{ $t('cancel') }}</span>
                            </button>
                            <button @click="save()" class="four columns">
                                <i class="fas fa-check"/> <span>{{ $t('ok') }}</span>
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import {dataService} from '../../../js/service/data/dataService'
    import {helpService} from "../../../js/service/helpService";
    import {i18nService} from "../../../js/service/i18nService";
    import Accordion from "../../components/accordion.vue"
    import InputEventList from "../../components/inputEventList.vue"
    import TestArea from "./testArea.vue"
    import './../../../css/modal.css';
    import {InputConfig} from "../../../js/model/InputConfig";
    import {inputEventHandler} from "../../../js/input/inputEventHandler";
    import {SequentialInput} from "../../../js/input/sequentialInput";
    import GlobalInputOptions from "./globalInputOptions.vue";

    export default {
        props: [],
        components: {GlobalInputOptions, Accordion, InputEventList, TestArea},
        data: function () {
            return {
                inputConfig: null,
                metadata: null,
                InputConfig: InputConfig,
                error: '',
                errorInputs: [],
                seqInput: null,
                testOpen: false,
                selectedTestElement: null
            }
        },
        watch: {
            inputConfig: {
                handler: function(newConfig) {
                    if (this.testOpen) {
                        this.initTest(newConfig);
                    }
                },
                deep: true
            }
        },
        methods: {
            save() {
                if (!this.validateInputs()) {
                    return;
                }
                this.metadata.inputConfig = this.inputConfig;
                dataService.saveMetadata(this.metadata).then(() => {
                    this.$emit('close');
                });
            },
            cancel() {
                this.$emit('close');
            },
            openHelp() {
                helpService.openHelp();
            },
            validateInputs() {
                this.errorInputs = [];
                this.error = "";
                if (!this.inputConfig.seqEnabled) {
                    return true;
                }
                if (this.inputConfig.seqInputs.filter(input => input.label === InputConfig.NEXT_ELEMENT).length === 0) {
                    this.errorInputs.push(InputConfig.NEXT_ELEMENT);
                }
                if (this.inputConfig.seqInputs.filter(input => input.label === InputConfig.SELECT).length === 0) {
                    this.errorInputs.push(InputConfig.SELECT);
                }
                if (this.errorInputs.length > 0) {
                    this.error = i18nService.t('pleaseSpecifyInputModalities');
                    return false;
                }
                return true;
            },
            inputChanged() {
                if (this.error) {
                    this.validateInputs();
                }
            },
            resetInput() {
                this.$set(this.inputConfig, 'seqInputs', JSON.parse(JSON.stringify(InputConfig.DEFAULT_SEQ_INPUTS)));
                this.inputChanged();
            },
            initTest() {
                setTimeout(() => {
                    let thiz = this;
                    thiz.stopTest();
                    if (thiz.inputConfig.seqEnabled) {
                        thiz.seqInput = SequentialInput.getInstanceFromConfig(thiz.inputConfig, '.area-element-inner', {
                            selectionListener: (item) => {
                                thiz.selectedTestElement = item;
                            },
                            activeClass: 'active'
                        });
                        thiz.seqInput.start();
                    }
                }, 100);
            },
            stopTest() {
                if (this.seqInput) {
                    this.seqInput.destroy();
                }
            }
        },
        mounted () {
            let thiz = this;
            inputEventHandler.pauseAll();
            dataService.getMetadata().then(metadata => {
                thiz.metadata = JSON.parse(JSON.stringify(metadata));
                thiz.inputConfig = JSON.parse(JSON.stringify(metadata.inputConfig));
            });
            helpService.setHelpLocation('04_input_options', '#sequential-input');
        },
        beforeDestroy() {
            helpService.revertToLastLocation();
            this.stopTest();
            inputEventHandler.resumeAll();
        }
    }
</script>

<style scoped>
    .warn {
        margin-top: 2em;
    }
</style>