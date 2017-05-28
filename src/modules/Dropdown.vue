<template>
    <div
        v-bind:class="[
            {visible},
            'ui',
            {active},
            {search},
            {selection},
            'dropdown'
        ]" v-on:click.self="toggle"
        v-on:keyup.down="selectNext"
        v-bind:tabindex="search ? '' : 0"
        v-click-outside="hide">
        <input type="hidden" v-bind:name="name" v-if="selection">
        <i class="dropdown icon" v-on:click="!search && isVisible ? hide() : show()"></i>
        <input type="search" class="search" autocomplete="off" tabindex="0"
            ref="search"
            v-model="term"
            v-on:focus="show"
            v-on:blur="hide"
            v-on:keyup.down="selectNext"
            v-if="search">
        <div v-bind:class="[
            { 'default' : typeof selected.value == 'undefined' },
            'text',
            { filtered },
        ]">{{ selected.text || selected.value || placeholder }}</div>
            <transition
                v-on:before-enter="startAnimating"
                v-on:after-enter="stopAnimating"
                enter-active-class="animating slide down in"
                leave-active-class="animating slide down out"
                v-on:after-leave="visible = false">
                <div
                    v-bind:class="[{visible}, 'transition', 'menu']"
                    tabindex="-1"
                    v-show="isVisible">
                    <slot>
                        <template v-show="$items.length > 0">
                            <div
                                v-bind:class="[
                                    { 'disabled' : item.disabled },
                                    { 'active selected' : selected == item && !disabled },
                                    'item'
                                ]"
                                v-bind:tabindex="(index + 2) * -1"
                                v-on:focus="setSelected(item)"
                                v-for="(item, index) in $items">
                                {{ item.name ? item.name : item.value }}
                            </div>
                        </template>
                        <div class="message" v-show="$items.length < 1">No results found.</div>
                    </slot>
                </div>
            </transition>
    </div>
</template>

<script>
    import ClickOutside from '../mixins/commons/click-outside'
    import Input from '../mixins/commons/input'
    import Diacritics from 'diacritics'
    import Arrai from '../mixins/commons/arrai.js'

    export default {
        mixins: [ClickOutside, Input, Arrai],
        props: {
            items: {
                type: Array,
                default: () => [],
            },
            selection: {
                type: Boolean,
                default() {
                    return typeof this.$vnode.componentOptions.propsData.search != 'undefined'
                        && this.$vnode.componentOptions.propsData.search == true
                },
            },
            search: {
                type: Boolean,
                default: false,
            },
            fullTextSearch: {
                type: Boolean,
                default: false,
            },
        },
        data() {
            return {
                /* Toggles active class */
                active: false,
                /* Toggles visible class */
                visible: false,
                /* Toggles element visibility */
                isVisible: false,
                /* Flags animation */
                animating: false,
                /* Search term */
                term: '',
                /* Test */
                selected: this.value,
            }
        },
        computed: {
            filtered() {
                return this.term.length > 0
            },
            $items() {
                var items = []
                var sanitized

                for (var i = 0; i < this.items.length; i++) {
                    /* Makes it easier to work */
                    var item = this.items[i];

                    /* Checks if it is a object */
                    if (this.isObject(item)) {
                            /* Checks for a value property */
                            if (typeof item.value == 'undefined' || item.value.length < 1) {
                                /* Show warning on JavaScript console */
                                console.warn(`Object must have a value property. Schema:
{
    // name displayed in dropdown
    "name"  : "Beyoncé Knowles",
    // selected value
    "value" : "${Math.floor(Math.random() * 10)}" // User ID,
    // name displayed after selection (optional)
    "text"  : "Beyoncé"
    // array of strings to test using user's input as a regexp (optional)
    "disabled"  : ['Beyonce Knowles'] // no diacritcs
    // whether field should be displayed as disabled (optional)
    "disabled"  : false
}
Ignoring item: `, item)

                                /* Skip this item */
                                continue
                            }

                            /* Put value in item name if it is not defined */
                            // item.name = ((typeof item.name == 'undefined') || (item.name.length < 1)) ? item.value : item.name
                            /* Put name in item text if it is not defined */
                            // item.text = (typeof item.text == 'undefined' || item.text.length < 1) ? item.name : item.text
                            /* Disabled's default value is false */
                            // item.disabled = (typeof item.disabled == 'undefined' || typeof item.disabled != 'boolean') ? false : item.disabled
                            /* Put all proprerties in item search if it is not defined */
                            // item.search = (typeof item.search == 'undefined' || item.search.length < 1) ? [item.name, item.text, item.value] : item.search

                            sanitized = item
                    } else {
                        /* Working with basics */
                        sanitized = {
                            value: item,
                        }
                    }

                    if (this.filtered) {
                        /* Nothing is selected */
                        this.selected = {}

                        /* Subjects for regexp */
                        var subjects = this.arrayFlatten([
                            sanitized.name,
                            sanitized.text,
                            sanitized.search,
                        ])

                        if (this.fullTextSearch) {
                            subjects.unshift(sanitized.value)
                        }

                        /* Escape user input */
                        var safe = this.term.split('').map(function(string) {
                            return string.replace(/[-\/\\^$*+?.()|[\]{}]/g, '\\$&')
                        })

                        /* Enable full search text or not */
                        var regexp = new RegExp(this.fullTextSearch ? safe.join('.*') : ('^' + safe.join('')), 'i')

                        /* If anything found */
                        if (regexp.test(subjects)) {
                            items.push(sanitized)
                        }
                    } else {
                        if (sanitized.value == this.value) {
                            this.selected = sanitized;
                        }

                        items.push(sanitized)
                    }
                }

                return items
            }
        },
        watch: {
            isVisible(newVal) {
                this.active = newVal
            },
            selected(newVal, oldVal) {
                if (newVal != oldVal && typeof newVal.value != 'undefined') {
                    this.$emit('input', newVal.value)
                    this.term = ''
                }
            }
        },
        methods: {
            toggle() {
                if (!this.animating) {
                    this.isVisible ? this.hide() : this.show()
                }

                console.log('toggle')
            },
            show() {
                this.isVisible = true
                if (this.search) {
                    this.$refs.search.focus()
                }

                console.log('show')
            },
            hide() {
                if (!this.fullTextSearch && this.filtered && this.$items.length > 0) {
                    this.selected = this.$items[0];
                }
                
                this.isVisible = false

                console.log('hide')
            },
            clear() {
                this.$emit('input', null)
            },
            restoreDefaults() {
            },
            setSelected(value) {
                this.selected = value
                if (!value.disabled) {
                    this.hide()
                }
            },
            selectNext() {
            },
            getText() {
            },
            getValue() {
            },
            startAnimating() {
                this.visible = this.isVisible
                this.animating = true
            },
            stopAnimating() {
                this.visible = this.isVisible
                this.animating = false
            },
            isObject(instance) {
                return !Array.isArray(instance) && instance === Object(instance) 
            },
            tmp() {
                console.log(this.$data)
            }
        },
        created() {
            if (typeof this.value == 'undefined' || this.value === null) {
                this.selected = {}
            }
        }
        /*mounted() {
            console.log(this.month)
        }*/
    }
</script>