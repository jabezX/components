<template>
    <dl-stepper-container
        v-if="isOpen"
        :id="uuid"
        class="dl-stepper-container"
        :style="cssVars"
        :transition="withTransition"
        :duration="transitionDuration"
    >
        <dl-stepper-header
            :header-title="headerTitle"
            :hide-close-button="hideCloseButton"
            @close="closeStepper"
        >
            <slot name="header-title" />
        </dl-stepper-header>
        <div class="dl-stepper-content">
            <dl-stepper-sidebar
                :steps="steps"
                :bg-color="bgColor"
                :hide="hide"
                :disabled="disabled"
                @step-click="$emit('set-step', $event)"
            />
            <div class="dl-stepper-content dl-stepper-content--slot">
                <dl-stepper-content
                    v-if="state"
                    :title="contentTitle"
                    :error="state.error"
                    :hide="hide"
                    :completed="state.completed"
                >
                    <template #header>
                        <slot
                            v-if="!isEmpty"
                            name="content-header"
                            :state="state"
                        />
                    </template>
                    <slot
                        v-if="!isEmpty"
                        :name="state.value"
                        :state="state"
                    />
                    <dl-empty-state
                        v-if="isEmpty && emptyStateProps"
                        v-bind="emptyStateProps"
                    >
                        <template
                            v-for="(_, slot) in $slots"
                            #[slot]="props"
                        >
                            <slot
                                :name="slot"
                                v-bind="props"
                            />
                        </template>
                    </dl-empty-state>
                </dl-stepper-content>
                <dl-stepper-footer
                    :finished="isDone"
                    :done-button-label="doneButtonLabel"
                    :next-button-label="nextButtonLabel"
                    :prev-button-label="prevButtonLabel"
                    :disabled-next-step="disabledNextStep"
                    :disabled-prev-step="disabledPrevStep"
                    :prev-step-disabled-tooltip="prevStepDisabledTooltip"
                    :next-step-disabled-tooltip="nextStepDisabledTooltip"
                    @next="$emit('next')"
                    @prev="$emit('prev')"
                    @done="$emit('done')"
                    @close="closeStepper"
                >
                    <template #footer>
                        <slot name="footer" />
                    </template>
                </dl-stepper-footer>
            </div>
        </div>
    </dl-stepper-container>
</template>
<script lang="ts">
import { defineComponent, PropType } from 'vue-demi'
import DlStepperContainer from './components/DlStepperContainer.vue'
import DlStepperHeader from './components/DlStepperHeader.vue'
import DlStepperFooter from './components/DlStepperFooter.vue'
import DlStepperSidebar from './components/DlStepperSidebar.vue'
import DlStepperContent from './components/DlStepperContent.vue'
import DlEmptyState from '../../basic/DlEmptyState/DlEmptyState.vue'
import { DlEmptyStateProps } from '../../basic/DlEmptyState/types'
import { StepState } from './models/interfaces'
import { Step } from './models'
import { getColor } from '../../../utils'
import { v4 } from 'uuid'

export default defineComponent({
    components: {
        DlStepperHeader,
        DlStepperFooter,
        DlStepperSidebar,
        DlStepperContent,
        DlStepperContainer,
        DlEmptyState
    },
    model: {
        prop: 'modelValue',
        event: 'update:model-value'
    },
    props: {
        steps: {
            type: Array as PropType<Step[]>,
            default: () => [] as Step[]
        },
        nextStep: {
            type: Object as PropType<StepState | null>,
            default: () => null as StepState | null
        },
        prevStep: {
            type: Object as PropType<StepState | null>,
            default: () => null as StepState | null
        },
        state: {
            type: Object as PropType<StepState>,
            default: null
        },
        modelValue: {
            type: Boolean,
            default: false
        },
        headerTitle: {
            type: String,
            required: false,
            default: ''
        },
        contentTitle: {
            type: String,
            default: ''
        },
        doneButtonLabel: {
            type: String,
            required: false,
            default: 'Done'
        },
        width: {
            type: String,
            required: false,
            default: '700px'
        },
        bgColor: {
            type: String,
            required: false,
            default: 'dl-color-fill-third'
        },
        withTransition: Boolean,
        transitionDuration: {
            type: Number,
            required: false,
            default: 0.3
        },
        disabledNextStep: Boolean,
        disabledPrevStep: Boolean,
        isDone: Boolean,
        hideCloseButton: Boolean,
        disabled: { type: Boolean, default: false },
        isEmpty: Boolean,
        emptyStateProps: {
            type: Object as PropType<DlEmptyStateProps>,
            default: null
        }
    },
    emits: ['update:model-value', 'done', 'next', 'prev', 'set-step', 'close'],
    data() {
        return {
            uuid: `dl-stepper-${v4()}`,
            isOpen: this.modelValue
        }
    },
    computed: {
        hide(): boolean {
            return this.isEmpty
        },
        nextButtonLabel(): string {
            return this.nextStep?.title ?? null
        },
        prevButtonLabel(): string {
            return this.prevStep?.title ?? null
        },
        nextStepDisabledTooltip(): string {
            return this.disabledNextStep
                ? this.nextStep?.disabledTooltip ?? ''
                : ''
        },
        prevStepDisabledTooltip(): string {
            return this.disabledPrevStep
                ? this.prevStep?.disabledTooltip ?? ''
                : ''
        },
        cssVars(): Record<string, string | number> {
            return {
                '--dl-stepper-width': this.width,
                '--dl-stepper-bg': getColor(this.bgColor, 'dl-color-fill-third')
            }
        }
    },
    watch: {
        modelValue(newVal: boolean) {
            this.isOpen = newVal
            if (newVal) document.documentElement.style.overflow = 'hidden'
            else document.documentElement.style.overflow = 'auto'
        }
    },
    methods: {
        closeStepper() {
            this.$emit('close')
            this.$emit('update:model-value', false)
        }
    }
})
</script>
<style lang="scss" scoped>
.dl-stepper-container {
    height: 100%;
    overflow: auto;
    width: var(--dl-stepper-width);
    display: flex;
    flex-direction: column;
    background-color: var(--dl-color-panel-background);
    border: 1px solid var(--dl-color-separator);
    border-radius: 2px;
}

.dl-stepper-content {
    display: flex;
    flex-grow: 1;
    position: relative;

    &--slot {
        display: flex;
        flex-direction: column;
        overflow: auto;
    }
}
</style>
