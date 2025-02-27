<template>
    <div
        :id="uuid"
        :style="cssVars"
        class="root-container"
        :class="{
            'root-container--s': isSmall,
            [identifierClass]: true,
            'fit-content': fitContent
        }"
    >
        <div
            v-show="!!title.length || !!tooltip.length"
            :class="{
                'dl-select__title-container': true,
                'dl-select__title-container--s': isSmall
            }"
        >
            <label
                v-show="!!title.length"
                class="dl-select__title"
            >
                {{ title
                }}<span
                    v-show="required"
                    :class="asteriskClasses"
                > *</span>
                {{ !required && optional ? ' (Optional)' : null }}
            </label>
            <span v-show="!!tooltip.length">
                <dl-icon
                    icon="icon-dl-info"
                    :inline="false"
                    class="tooltip-icon"
                    size="12px"
                />
                <dl-tooltip>
                    {{ tooltip }}
                </dl-tooltip>
            </span>
        </div>
        <div
            v-show="!!topMessage.length && !isSmall"
            class="top-message-container"
        >
            <dl-info-error-message
                v-show="!!topMessage.length"
                position="above"
                :value="topMessage"
            />
        </div>
        <div
            class="select-wrapper"
            tabindex="0"
            :style="placeholderStyles"
        >
            <div
                ref="select"
                :class="selectClasses"
            >
                <div
                    v-show="hasPrepend || searchable"
                    :class="[
                        ...adornmentClasses,
                        'adornment-container--pos-left'
                    ]"
                >
                    <slot name="prepend">
                        <dl-icon
                            v-if="searchable"
                            icon="icon-dl-search"
                            :size="iconSize"
                            color="dl-color-lighter"
                        />
                    </slot>
                </div>
                <input
                    v-if="searchable"
                    ref="searchInput"
                    class="select-search-input"
                    :style="!isExpanded ? 'display: none;' : 'width: 100%;'"
                    :disabled="disabled"
                    :readonly="readonly"
                    @input="handleSearchInput"
                    @focus="handleSearchFocus"
                    @blur="handleSearchBlur"
                >
                <dl-tooltip v-if="disabled && disabledTooltip">
                    {{ disabledTooltip }}
                </dl-tooltip>
                <div
                    v-if="hasSelectedSlot"
                    style="width: 100%"
                >
                    <slot
                        v-if="searchable ? !isExpanded : true"
                        :opt="selectedOption"
                        name="selected"
                    />
                </div>
                <template v-if="!hasSelectedSlot">
                    <span
                        v-show="
                            (multiselect && !searchable) ||
                                (multiselect && searchable && !isExpanded)
                        "
                        class="root-container--placeholder"
                    >
                        <template v-if="allFiltersModel">
                            {{ computedAllItemsLabel }}
                        </template>
                        <template v-else>
                            {{ filterSelectLabel }}
                        </template>
                    </span>
                    <span
                        v-show="
                            (!multiselect && !searchable) ||
                                (!multiselect && searchable && !isExpanded)
                        "
                        class="selected-label"
                    >
                        {{ getLabel(selectedOption) }}
                    </span>
                </template>
                <div
                    :class="[
                        ...adornmentClasses,
                        `adornment-container--pos-right${
                            withoutDropdownIconPadding
                                ? ' adornment-container--pos-right-without_padding'
                                : ''
                        }`
                    ]"
                >
                    <dl-icon
                        v-if="clearable && hasSelection"
                        class=".dl-select__clear-button"
                        icon="icon-dl-close"
                        :size="withoutBorders ? '10px' : '12px'"
                        style="margin-right: 3px; cursor: pointer"
                        @click.prevent.stop="clearSelection"
                    />
                    <dl-icon
                        :icon="dropdownIcon"
                        :color="chevronIconColor"
                        class="expand-icon"
                        :inline="false"
                        :size="withoutBorders ? '12px' : '16px'"
                        :class="{ expanded: isExpanded }"
                    />
                </div>
            </div>
            <dl-menu
                ref="menu"
                v-model="isExpanded"
                fit-container
                :fit-content="fitContent"
                square
                no-focus
                :offset="[0, 3]"
                style="border-radius: 0"
                :disabled="disabled || readonly"
                :arrow-nav-items="options"
                :max-height="dropdownMaxHeight"
                :trigger-percentage="triggerPercentage"
                @show="onMenuOpen"
                @hide="closeMenu"
                @highlightedIndex="setHighlightedIndex"
                @handleSelectedItem="handleSelectedItem"
            >
                <dl-list
                    class="select-list"
                    :padding="false"
                >
                    <dl-list-item v-if="noOptions">
                        <dl-item-section color="dl-color-medium">
                            <slot name="no-options">
                                No options
                            </slot>
                        </dl-item-section>
                    </dl-list-item>
                    <dl-list-item v-if="hasBeforeOptions && !noOptions">
                        <dl-item-section color="dl-color-medium">
                            <slot name="before-options" />
                        </dl-item-section>
                    </dl-list-item>
                    <dl-select-option
                        v-if="showAllItems"
                        :multiselect="multiselect"
                        :with-wave="withWave"
                        clickable
                        :model-value="allFiltersModel"
                        :count="totalCount"
                        :highlight-selected="highlightSelected"
                        total-items
                        @update:model-value="selectAll"
                        @depth-change="handleDepthChange"
                    >
                        <slot name="all-items">
                            {{ computedAllItemsLabel }}
                        </slot>
                    </dl-select-option>

                    <!-- Virtual scroll -->
                    <dl-virtual-scroll
                        v-if="optionsCount > MAX_ITEMS_PER_LIST"
                        v-slot="{ item, index }"
                        :items="filteredOptions"
                        :virtual-scroll-item-size="28"
                        :virtual-scroll-sticky-size-start="28"
                        :virtual-scroll-sticky-size-end="20"
                        separator
                    >
                        <dl-select-option
                            :key="getKeyForOption(item)"
                            clickable
                            :multiselect="multiselect"
                            :class="{
                                selected:
                                    item === selectedOption && highlightSelected
                            }"
                            :style="
                                index === highlightedIndex
                                    ? 'background-color: var(--dl-color-fill)'
                                    : ''
                            "
                            :with-wave="withWave"
                            :model-value="modelValue"
                            :value="getOptionValue(item)"
                            :highlight-selected="highlightSelected"
                            :count="getOptionCount(item)"
                            :children="getOptionChildren(item)"
                            :capitalized="capitalizedOptions"
                            :readonly="isReadonlyOption(item)"
                            @update:model-value="handleModelValueUpdate"
                            @click="selectOption(item)"
                            @selected="handleSelected"
                            @deselected="handleDeselected"
                        >
                            <slot
                                name="option"
                                :opt="item"
                            >
                                <span
                                    class="inner-option"
                                    v-html="getOptionHtml(item)"
                                />
                            </slot>
                        </dl-select-option>
                    </dl-virtual-scroll>
                    <!-- Else normal list -->
                    <div v-else>
                        <dl-select-option
                            v-for="(option, optionIndex) in filteredOptions"
                            :key="getKeyForOption(option)"
                            clickable
                            :multiselect="multiselect"
                            :class="{
                                selected:
                                    option === selectedOption &&
                                    highlightSelected
                            }"
                            :style="
                                optionIndex === highlightedIndex
                                    ? 'background-color: var(--dl-color-fill)'
                                    : ''
                            "
                            :with-wave="withWave"
                            :model-value="modelValue"
                            :value="getOptionValue(option)"
                            :highlight-selected="highlightSelected"
                            :count="getOptionCount(option)"
                            :children="getOptionChildren(option)"
                            :capitalized="capitalizedOptions"
                            :readonly="isReadonlyOption(option)"
                            @update:model-value="handleModelValueUpdate"
                            @click="selectOption(option)"
                            @selected="handleSelected"
                            @deselected="handleDeselected"
                            @depth-change="handleDepthChange"
                        >
                            <slot
                                :opt="option"
                                name="option"
                            >
                                <dl-ellipsis>
                                    <span
                                        class="inner-option"
                                        v-html="getOptionHtml(option)"
                                    />
                                </dl-ellipsis>
                            </slot>
                        </dl-select-option>
                    </div>
                    <dl-list-item v-if="hasAfterOptions && !noOptions">
                        <dl-item-section>
                            <slot name="after-options" />
                        </dl-item-section>
                    </dl-list-item>
                </dl-list>
            </dl-menu>
        </div>
        <div
            v-show="!isSmall && (!!infoMessage.length || !!errorMessage.length)"
            class="bottom-message-container"
        >
            <dl-info-error-message
                v-show="!!infoMessage.length && !error"
                position="below"
                :value="infoMessage"
            />
            <dl-info-error-message
                v-show="error && !!errorMessage.length"
                position="below"
                error
                :value="errorMessage"
            />
        </div>
    </div>
</template>

<script lang="ts">
import { InputSizes, TInputSizes } from '../../../utils/input-sizes'
import { DlListItem } from '../../basic'
import { DlTooltip } from '../../shared'
import { DlList, DlIcon, DlMenu } from '../../essential'
import {
    DlInfoErrorMessage,
    DlItemSection,
    DlVirtualScroll
} from '../../shared'
import { DlEllipsis } from '../../essential'
import { defineComponent, PropType, ref } from 'vue-demi'
import {
    getLabel,
    getIconSize,
    optionsValidator,
    DlSelectOptionType,
    getLabelOfSelectedOption,
    getCaseInsensitiveInput
} from './utils'
import DlSelectOption from './components/DlSelectOption.vue'
import { isEqual } from 'lodash'
import { getColor } from '../../../utils'
import { v4 } from 'uuid'

export default defineComponent({
    components: {
        DlIcon,
        DlInfoErrorMessage,
        DlList,
        DlItemSection,
        DlListItem,
        DlMenu,
        DlTooltip,
        DlSelectOption,
        DlVirtualScroll,
        DlEllipsis
    },
    model: {
        prop: 'modelValue',
        event: 'update:model-value'
    },
    props: {
        modelValue: { type: [String, Object, Array, Number], default: '' },
        size: {
            type: String as PropType<TInputSizes>,
            default: InputSizes.l
        },
        withWave: Boolean,
        alignRight: { type: Boolean, default: false },
        allItemsOption: { type: Boolean, default: false },
        allItemsOptionLabel: { type: String, default: null },
        placeholder: { type: String, default: null },
        removableSelection: { type: Boolean, default: false },
        width: { type: String, default: '100%' },
        withoutBorders: { type: Boolean, default: false },
        title: { type: String, default: '' },
        searchable: { type: Boolean, default: false },
        customFilter: { type: Boolean, default: false },
        required: { type: Boolean, default: false },
        optional: { type: Boolean, default: false },
        fitContent: Boolean,
        tooltip: { type: String, default: '' },
        highlightSelected: { type: Boolean, default: false },
        type: { type: String, default: 'text' },
        multiselect: { type: Boolean, default: false },
        dropdownIcon: { type: String, default: 'icon-dl-down-chevron' },
        topMessage: { type: String, default: '' },
        redAsterisk: { type: Boolean, default: false },
        infoMessage: { type: String, default: '' },
        errorMessage: { type: String, default: '' },
        error: { type: Boolean, default: false },
        disabled: { type: Boolean, default: false },
        readonly: { type: Boolean, default: false },
        emitValue: { type: Boolean, default: false }, // We emit the value from the option and compare with it as a modelvalue
        options: {
            type: Array as PropType<DlSelectOptionType[]>,
            default: (): DlSelectOptionType[] => [],
            validator: optionsValidator
        },
        capitalizedOptions: { type: Boolean, default: false },
        withoutDropdownIconPadding: { type: Boolean, default: false },
        clearable: { type: Boolean, default: false },
        dropdownMaxHeight: { type: String, default: '30vh' },
        preserveSearch: { type: Boolean, default: false },
        disabledTooltip: { type: String, default: 'Disabled' },
        /**
         * the % of the select element to display the menu
         */
        triggerPercentage: {
            type: Number,
            default: 1
        }
    },
    emits: [
        'search-focus',
        'search-blur',
        'filter',
        'change',
        'search-input',
        'update:model-value',
        'before-show',
        'before-hide',
        'show',
        'hide',
        'selected',
        'deselected'
    ],
    setup(props, { emit }) {
        const isExpanded = ref(false)
        const selectedIndex = ref(-1)
        const highlightedIndex = ref(-1)
        const isEmpty = ref(true)
        const searchTerm = ref('')
        const searchInputValue = ref('')
        const MAX_ITEMS_PER_LIST = 100 // HARDCODED - max items per list before virtual scroll

        const setHighlightedIndex = (value: any) => {
            highlightedIndex.value = value
        }
        const handleModelValueUpdate = (val: any) => {
            emit('update:model-value', val)
            emit('change', val)
            emit('selected', val)
        }

        return {
            uuid: `dl-select-${v4()}`,
            MAX_ITEMS_PER_LIST,
            isEmpty,
            isExpanded,
            highlightedIndex,
            selectedIndex,
            setHighlightedIndex,
            handleModelValueUpdate,
            searchTerm, // todo: merge this sometime
            searchInputValue
        }
    },
    computed: {
        hasSelection(): boolean {
            return !!this.modelValueLength || this.selectedIndex !== -1
        },
        filteredOptions(): DlSelectOptionType[] {
            if (this.customFilter || this.searchTerm === '') {
                return this.options
            }

            return this.options.filter((option) => {
                const label = getLabel(option)
                return (
                    label &&
                    label
                        .toLowerCase()
                        .includes(this.searchTerm.toLowerCase().trim())
                )
            })
        },
        optionsCount(): number {
            return this.options?.length ?? 0
        },
        identifierClass(): string {
            return `dl-select-${this.title}-${
                this.placeholder ?? ''
            }`.replaceAll(' ', '-')
        },
        showAllItems: {
            get(): boolean {
                return this.multiselect && this.allItemsOption && this.isEmpty
            },
            set(val: boolean) {
                this.isEmpty = val
            }
        },
        modelValueLength(): number {
            if (
                typeof this.modelValue !== 'string' &&
                !Array.isArray(this.modelValue)
            ) {
                return 0
            }
            return this.modelValue?.length ?? 0
        },
        allFiltersModel(): boolean {
            if (Array.isArray(this.modelValue)) {
                const options = this.options.some((opt) =>
                    this.getOptionChildren(opt)
                )
                return (
                    this.modelValue?.length === this.options.length && !options
                )
            }
            return false
        },
        filterSelectLabel(): string {
            if (this.modelValueLength === 1) {
                const valueToSearch = (this.modelValue as any)[0]
                return getLabelOfSelectedOption(valueToSearch, this.options)
            }
            return this.modelValueLength > 0
                ? `${this.modelValueLength} Selected Items`
                : this.computedPlaceholder
        },
        computedAllItemsLabel(): string {
            return this.allItemsOptionLabel || 'All Items'
        },
        isModelValuePrimitiveType(): boolean {
            return this.isPrimitiveValue(this.modelValue)
        },
        totalCount(): number {
            let total = 0
            if (typeof this.options[0] !== 'string') {
                this.options.forEach((option: any) => {
                    if (typeof option === 'object' && option!.count) {
                        total += option!.count as number
                    }
                })
            }

            return total
        },
        iconSize(): string {
            return getIconSize(this.size)
        },
        hasOptionSlot(): boolean {
            return !!this.$slots.option
        },
        hasAllItemsSlot(): boolean {
            return !!this.$slots['all-items']
        },
        hasSelectedSlot(): boolean {
            return !!this.$slots.selected
        },
        computedPlaceholder(): string {
            return this.placeholder || 'Select option'
        },
        placeholderStyles(): Record<string, string> {
            if (this.disabled) {
                return
            }
            return {
                '--placeholder-color': getColor(
                    this.modelValueLength > 0 || this.selectedIndex !== -1
                        ? 'dl-color-darker'
                        : 'dl-color-lighter'
                )
            }
        },
        selectedOption(): string | Record<string, string | number> | number {
            return this.selectedIndex === -1
                ? this.computedPlaceholder
                : this.options[this.selectedIndex]
        },
        hasBeforeOptions(): boolean {
            return !!this.$slots['before-options']
        },
        hasAfterOptions(): boolean {
            return !!this.$slots['after-options']
        },
        noOptions(): boolean {
            if (Array.isArray(this.options)) {
                if (this.customFilter) {
                    return this.options.length === 0
                }

                return this.filteredOptions.length === 0
            }
            return true
        },
        selectClasses(): string[] {
            const classes = ['dl_select__select']

            classes.push(`dl_select__select--${this.size}`)
            classes.push('dl_select__select--append')

            if (this.withoutDropdownIconPadding) {
                classes.push('dl_select__select--append-without_padding')
            }
            if (this.selectedIndex !== -1) {
                classes.push('dl_select__select--has-selection')
            }
            if (this.alignRight) {
                classes.push('dl_select__select--align-right')
            }
            if (this.withoutBorders) {
                classes.push('dl_select__select--without-border')
            }
            if (this.withoutBorders && this.hasPrepend) {
                classes.push('dl_select__select--without-border__with-prepend')
            }
            if (this.hasPrepend || this.searchable) {
                classes.push('dl_select__select--prepend')
                classes.push('dl_select__select--both-adornments')
            }
            if (this.error) {
                classes.push('dl_select__select--error')
            }
            if (this.disabled) {
                classes.push('dl_select__select--disabled')
            }
            if (this.readonly) {
                classes.push('dl_select__select--readonly')
            }
            if (this.isExpanded) {
                classes.push('dl_select__select--focused')
            }

            return classes
        },
        cssVars(): Record<string, string> {
            return {
                '--dl-select-width': this.width,
                '--dl-select-expand-icon-width': this.withoutDropdownIconPadding
                    ? '16px'
                    : '28px',
                '--dl-menu-scrollbar-width': '10px'
            }
        },
        asteriskClasses(): string[] {
            const classes = ['required-asterisk']
            if (this.redAsterisk) {
                classes.push('required-asterisk--red')
            }
            return classes
        },
        adornmentClasses(): string[] {
            const classes = ['adornment-container']
            classes.push(`adornment-container--${this.size}`)
            return classes
        },
        isSmall(): boolean {
            return (
                this.size === (InputSizes.s as TInputSizes) ||
                this.size === (InputSizes.small as TInputSizes)
            )
        },
        hasPrepend(): boolean {
            return !!this.$slots.prepend && !this.isSmall
        },
        chevronIconColor(): string {
            return `${this.disabled ? 'dl-color-disabled' : null}`
        }
    },
    watch: {
        isExpanded(newVal: boolean) {
            if (this.searchable) {
                if (newVal) {
                    const inputRef = this.$refs.searchInput as HTMLInputElement
                    this.$nextTick(() => {
                        inputRef?.focus({})
                    })
                }
            }
        },
        modelValue() {
            this.setSelectedIndex()
        },
        emitValue() {
            this.setSelectedIndex()
        }
    },
    beforeMount() {
        this.setSelectedIndex()
    },
    methods: {
        handleDepthChange() {
            // todo: remove this hack
            setTimeout(() => {
                this.$nextTick(() => {
                    // @ts-ignore
                    this.$refs.menu?.updatePosition()
                })
            }, 100)
        },
        isPrimitiveValue(option: any): boolean {
            return (
                typeof option === 'string' ||
                typeof option === 'number' ||
                typeof option === 'boolean'
            )
        },
        setSelectedIndex() {
            if (!this.modelValue) {
                this.selectedIndex = -1
                return
            }

            if (this.emitValue) {
                this.selectedIndex = this.options.findIndex(
                    (
                        option:
                            | string
                            | Record<string, string | number>
                            | number
                    ) =>
                        isEqual(
                            (option as any).value,
                            this.modelValue as string | number
                        )
                )
                return
            }

            if (this.isModelValuePrimitiveType) {
                this.selectedIndex = this.options.findIndex(
                    (
                        option:
                            | string
                            | Record<string, string | number>
                            | number
                    ) => option === this.modelValue
                )
                return
            }

            this.selectedIndex = this.options.findIndex(
                (option: string | Record<string, string | number> | number) =>
                    isEqual(option, this.modelValue)
            )
        },
        handleSelectedItem(value: DlSelectOptionType) {
            this.selectOption(value)
        },
        getOptionValue(option: any) {
            return option?.value ?? option
        },
        getOptionLabel(option: any) {
            return getLabel(option) ?? this.getOptionValue(option)
        },
        getOptionChildren(option: any) {
            if (typeof option === 'string') {
                return null
            }
            return option?.children && option?.children?.length
                ? option?.children
                : null
        },
        isReadonlyOption(option: any) {
            return !!option?.readonly
        },
        getOptionCount(option: any) {
            return option?.count ?? null
        },
        getKeyForOption(
            option: string | number | Record<string, string | number>
        ) {
            if (typeof option === 'string' || typeof option === 'number') {
                return option
            }
            return option.key ?? option.label
        },
        handleMenuTrigger(val: boolean) {
            this.isExpanded = val
        },
        selectAll(val: string[] | boolean, e: Event): void {
            e.preventDefault()
            e.stopPropagation()
            let toEmit: string[] = []
            if (
                !['string', 'number'].includes(typeof this.options[0]) &&
                !this.isModelValuePrimitiveType &&
                (this.modelValue as any)?.length < this.options.length
            ) {
                toEmit = this.options.map((option: any) => option!.value)
            }
            this.$emit('update:model-value', toEmit)
            this.$emit('change', toEmit)
        },
        clearSelection(): void {
            let toEmit: any[] | string = []
            if (this.isModelValuePrimitiveType) {
                toEmit = ''
            }

            this.$emit('update:model-value', toEmit)
            this.$emit('change', toEmit)
            this.selectedIndex = -1
            this.closeMenu()
        },
        selectOption(selectedOption: DlSelectOptionType) {
            if (this.multiselect || this.isReadonlyOption(selectedOption)) {
                return
            }

            if (this.searchable) {
                const searchInput = this.$refs.searchInput as HTMLInputElement
                searchInput.value =
                    typeof selectedOption === 'string'
                        ? selectedOption
                        : (selectedOption as Record<string, any>)?.label
            }
            this.closeMenu()

            const valueToEmit =
                this.emitValue && !this.isPrimitiveValue(selectedOption)
                    ? (selectedOption as Record<string, any>)?.value
                    : selectedOption

            this.handleModelValueUpdate(valueToEmit)
        },
        handleSearchInput(e: Event): void {
            if (this.searchable) {
                if (this.customFilter) {
                    this.$emit(
                        'filter',
                        (e.target as HTMLInputElement).value.trim()
                    )
                } else {
                    this.searchTerm = (
                        e.target as HTMLInputElement
                    ).value.trim()
                }

                this.showAllItems =
                    (e.target as HTMLInputElement).value.trim() === ''

                this.$nextTick(() => {
                    const menu = this.$refs.menu as any
                    menu?.updatePosition()
                })
            }
            const searchValue = (e.target as HTMLInputElement).value
            this.searchInputValue = searchValue
            this.$emit('search-input', searchValue)
        },
        getOptionHtml(option: DlSelectOptionType) {
            const label = `${this.getOptionLabel(option)}`
            const toReplace = new RegExp(this.searchInputValue, 'gi')

            const highlightedHtml = label.replace(
                toReplace,
                `<span style="background: var(--dl-color-warning)">${getCaseInsensitiveInput(
                    label,
                    this.searchInputValue
                )}</span>`
            )
            const html = `<span>${highlightedHtml}</span>`

            return html
        },
        handleSearchBlur(e: Event): void {
            if (this.searchable) {
                const inputRef = this.$refs.searchInput as HTMLInputElement
                this.$nextTick(() => {
                    inputRef?.focus({})
                })
                this.$emit('search-blur', e)
            }
        },
        closeMenu(): void {
            this.$emit('before-hide')
            this.isExpanded = false
            this.$emit('hide')

            if (!this.preserveSearch) {
                const inputRef = this.$refs.searchInput as HTMLInputElement
                if (inputRef) inputRef.value = ''
                this.searchTerm = ''
                this.searchInputValue = ''
                this.$emit('filter', '')
            }
        },
        getLabel,
        onMenuOpen(): void {
            this.$emit('before-show')
            if (this.searchable) {
                const inputRef = this.$refs.searchInput as HTMLInputElement
                this.$nextTick(() => {
                    inputRef?.focus({})
                })
            }
            this.$emit('show')
        },
        handleSearchFocus(e: FocusEvent): void {
            this.$emit('search-focus', e)
        },
        onBlur(e: InputEvent): void {
            this.$emit('search-blur', e)
        },
        handleSelected(value: any) {
            this.$emit('selected', value)
        },
        handleDeselected(value: any) {
            this.$emit('deselected', value)
        }
    }
})
</script>

<style scoped lang="scss">
.root-container {
    width: var(--dl-select-width);
    &--s,
    &--small {
        display: flex;
        align-items: center;
    }
    &--placeholder {
        color: var(--dl-select-palceholder-color, --placeholder-color);
    }

    .dl-select__title-container {
        margin-bottom: 6px;
        display: flex;
        align-items: center;
        color: var(--dl-color-lighter);

        &--s,
        &--small {
            margin-bottom: 0;
            margin-right: 4px;
        }
    }

    .selected-label {
        color: var(--dl-select-palceholder-color, --placeholder-color);
    }

    .dl-select__title {
        color: var(--dl-color-medium);
        font-size: 12px;
        text-align: left;
        margin-right: 5px;
        white-space: nowrap;
    }

    .required-asterisk {
        color: var(--dl-color-medium);
        font-size: 12px;
        user-select: none;

        &--red {
            color: var(--dl-color-negative);
        }
    }

    .tooltip-icon {
        color: var(--dl-color-medium);
    }

    .top-message-container {
        display: flex;
        margin-bottom: 10px;
        text-align: start;
    }

    .select-wrapper {
        position: relative;
        width: 100%;
    }

    .dl_select__select {
        border: 1px solid var(--dl-color-separator);
        border-radius: 2px;
        cursor: pointer;
        color: var(--dl-color-darker);
        height: 12px;
        width: 100%;
        box-sizing: content-box;
        padding: 10px;
        outline: none;
        background: none;
        font-size: 12px;
        position: relative;
        display: flex;
        align-items: center;
        transition-property: border-color;
        transition-duration: 0.28s, 0.28s;
        transition-timing-function: ease, ease;
        transition-delay: 0s, 0s;

        &--align-right {
            justify-content: flex-end;
        }

        &--prepend {
            padding-left: 28px;
            width: calc(100% - 10px - 28px);
        }

        &--append {
            padding-right: var(--dl-select-expand-icon-width);
            width: calc(100% - 10px - var(--dl-select-expand-icon-width));
        }

        &--both-adornments {
            width: calc(100% - 28px - 28px);
        }

        &--m {
            padding-top: 7px;
            padding-bottom: 7px;
        }
        &--medium {
            padding-top: 7px;
            padding-bottom: 7px;
        }

        &--s {
            padding-top: 3px;
            padding-bottom: 3px;
            padding-left: 5px;
            padding-right: 5px;
            width: calc(100% - 10px);
        }
        &--small {
            padding-top: 3px;
            padding-bottom: 3px;
            padding-left: 5px;
            padding-right: 5px;
            width: calc(100% - 10px);
        }

        &--has-selection {
            color: var(--dl-color-medium);
        }

        &--error {
            border-color: var(--dl-color-negative);
        }
        &--without-border {
            border: none;
            width: calc(100% - var(--dl-select-expand-icon-width));
            padding: 0;
            padding-right: var(--dl-select-expand-icon-width);
            .adornment-container {
                height: 100%;
            }
            height: auto;

            &__with-prepend {
                padding-left: 30px;
                padding-top: 10px;
                padding-bottom: 10px;
                width: calc(80% - var(--dl-select-expand-icon-width));
            }
        }

        &::placeholder {
            color: var(--dl-color-lighter);
            opacity: 1;
        }

        &:hover {
            border-color: var(--dl-color-hover);
        }

        &--focused {
            border-color: var(--dl-color-secondary);
        }

        &--disabled {
            border-color: var(--dl-color-separator);
            color: var(--dl-color-disabled);
            cursor: not-allowed;

            &:hover {
                border-color: var(--dl-color-separator);
            }
            & input {
                pointer-events: none;
            }
        }

        &--readonly {
            border-color: var(--dl-color-separator);
            cursor: text;

            &:hover {
                border-color: var(--dl-color-separator);
            }
            & input {
                pointer-events: none;
            }
        }
    }

    .selected {
        background-color: var(--dl-color-fill);
    }

    //todo: This doesnt work because of portal.
    .select-list {
        padding: 5px 0;
        max-height: var(--dl-select-dropdown-max-height);
        overflow: auto;
    }

    .expand-icon {
        display: flex !important;
        justify-content: center !important;
        align-items: center;
        color: var(--dl-color-medium);
        transition-property: transform, -webkit-transform;
        transition-duration: 0.28s, 0.28s;
        transition-timing-function: ease, ease;
        transition-delay: 0s, 0s;
        &.expanded {
            transform: rotate(180deg);
        }
    }

    .adornment-container {
        display: flex;
        justify-content: center;
        align-items: center;
        position: absolute;
        width: 28px;

        &--l {
            height: 34px;
        }

        &--large {
            height: 34px;
        }

        &--m {
            height: 28px;
        }

        &--medium {
            height: 28px;
        }

        &--s {
            height: 20px;
        }

        &--small {
            height: 20px;
        }

        &--pos-left {
            top: 0;
            left: 0;
        }

        &--pos-right {
            top: 0;
            right: 0;
            &-without_padding {
                width: 14px;
            }
        }
    }

    .select-search-input {
        appearance: none;
        border: 0;
        outline: none;
        background: none;
        color: var(--dl-color-darker);
        font-family: 'Roboto', sans-serif;
        font-size: 12px;
        height: 14px;

        &.hidden {
            position: absolute;
            width: 1px;
            height: 1px;
            padding: 0;
            margin: -1px;
            overflow: hidden;
            clip: rect(0, 0, 0, 0);
            white-space: nowrap;
            border: 0;
        }
    }
    .bottom-message-container {
        display: flex;
        justify-content: space-between;
        text-align: left;
        padding-top: 3px;
    }
}
.fit-content > * {
    max-width: fit-content;
}
</style>
