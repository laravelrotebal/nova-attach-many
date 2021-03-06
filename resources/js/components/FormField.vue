<template>
    <default-field :field="field" :full-width-content="field.fullWidth" :show-help-text="false">
        <template slot="field">
            <div>
                <div v-if="field.showToolbar" class="flex relative">
                    <div v-if="preview" class="flex justify-center items-center absolute pin z-10 bg-white">
                        <h3>{{ __('Selected Items') }} ({{ selected.length  }})</h3>
                    </div>

<!--                    <div @click="selectAll" class="w-16 text-center flex justify-center items-center">
                        <fake-checkbox :checked="selectingAll" class="cursor-pointer"></fake-checkbox>
                    </div>-->

                    <div class="flex-1 flex items-center relative">
                        <input v-model="search" type="text" :placeholder="__('Search')" class="w-full form-control form-input form-input-bordered" :class="{'border-danger border': hasErrors}">
                        <span v-if="search" @click="clearSearch" class="pin-r font-sans font-bolder absolute pr-8 cursor-pointer text-black hover:text-80">x</span>
                    </div>
                </div>

                <div class="relative overflow-auto" :style="{ height: field.height, width: '99%', marginTop: '10px', maxHeight: '134px', minHeight: '44px' }">
                    <div v-if="loading" class="flex justify-center items-center absolute pin z-50 bg-white">
                        <loader class="text-60" />
                    </div>

                    <div v-else v-for="resource in resources" :key="resource.value" @click="toggle($event, resource)" class="flex py-3 cursor-pointer select-none hover:bg-30" :class="{'border-danger border': hasErrors}">
                        <div class="w-16 flex justify-center">
                            <fake-checkbox :checked="selected.includes(resource)" />
                        </div>
                        <span>{{ resource.display }}</span>
                    </div>
                </div>
            </div>

            <help-text class="error-text mt-2 text-danger" v-if="hasErrors">
                {{ firstError }}
            </help-text>

            <div class="help-text mt-3 w-full flex" :class="{ 'invisible': loading }">
                <span v-if="field.showCounts" class="pr-2 float-left border-60 whitespace-no-wrap" :class="{ 'border-r mr-2': field.helpText }">
                    {{ selected.length  }} / {{ available.length }}
                </span>

                <span class="float-left border-60" :class="{'border-r mr-2': field.showPreview }">
                    <help-text class="help-text" v-if="field.helpText"> {{ field.helpText }} </help-text>
                </span>

                <span v-if="field.showPreview" @click="togglePreview($event)" class="flex cursor-pointer select-none float-right">
                    <span class="pr-2">{{ __('Preview') }}</span>
                    <fake-checkbox class="flex" :checked="preview" />
                </span>
            </div>

        </template>
    </default-field>
</template>

<script>
import { FormField, HandlesValidationErrors } from 'laravel-nova'

export default {
    mixins: [FormField, HandlesValidationErrors],

    props: ['resourceName', 'resourceId', 'field'],

    data() {
        return {
            ajaxes: [],
            search: null,
            selected: [],
            selectingAll: false,
            available: [],
            displayed: [],
            preview: false,
            loading: true,
        }
    },
    methods: {
        setInitialValue() {

            let baseUrl = '/nova-vendor/nova-attach-many/';

            if(this.resourceId) {
                Nova.request(baseUrl + this.resourceName + '/' + this.resourceId + '/attachable/' + this.field.attribute)
                    .then((data) => {
                        this.selected = data.data.selected || [];
                        this.available = data.data.available || [];
                        this.displayed = this.selected.concat(this.available);
                        this.loading = false;
                    });
            }
            else {
                Nova.request(baseUrl + this.resourceName + '/attachable/' + this.field.attribute)
                    .then((data) => {
                        this.available = data.data.available || [];
                        this.displayed = this.available;
                        this.loading = false;
                    });
            }

        },

        fill(formData) {
            formData.append(this.field.attribute, this.value || [])
        },

        toggle(event, resource) {
            if(this.selected.includes(resource) ) {
                this.selected = this.selected.filter(selectedId => selectedId != resource);
            }
            else {
                this.selected.push(resource)
            }
        },

        selectAll() {
            var selected = this.selected;

            this.selectingAll = ! this.selectingAll;

            // search can return 0 results
            if(this.resources.length == 0) {
                return;
            }

            if(this.resources.length == 1 && this.selected == 1)
            {
                this.selected = [];
            }

            // add all resources
            if(! this.search && this.selectingAll) {
                selected = [];
                this.resources.forEach(resource => {
                    selected.push(resource.value)
                })
            }

            // remove all resources
            if(! this.search && ! this.selectingAll) {
                selected = [];
            }

            // append searched resources
            if(this.search && this.selectingAll) {
                this.resources.forEach(resource => {
                    selected.push(resource.value)
                })
            }

            // remove only searched items
            if(this.search && ! this.selectingAll) {

                let exludingIds = [];

                this.resources.forEach(resource => {
                    exludingIds.push(resource.value);
                })

                selected = selected.filter(id => exludingIds.includes(id) == false);
            }

            this.selected = selected;
        },

        clearSearch() {
            this.displayed = this.selected;
            this.selectingAll = false;
            this.search = null;
        },

        checkIfSelectAllIsActive() {

            if(this.resources.length === 0 || this.preview) {
                this.selectingAll = false; return;
            }

            let visibleAndSelected = [];

            this.resources.forEach(resource => {
                if(this.selected.includes(resource.value)) {
                    visibleAndSelected.push(resource.value);
                }
            })

            this.selectingAll = visibleAndSelected.length == this.resources.length;
        },

        togglePreview(event){
            this.preview = ! this.preview;
        }
    },
    computed: {
        resources: function() {
            if(this.preview) {
                return this.available.filter((resource) => {
                    return this.selected.includes(resource.value)
                });
            }

            if(this.search == null) {
                return this.displayed;
            }

            return this.displayed;
        },

        hasErrors: function() {
            return this.errors.errors.hasOwnProperty(this.field.attribute);
        },

        firstError: function() {
            return this.errors.errors[this.field.attribute][0]
        }
    },
    watch: {
        'search': {
            handler: function(search) {
                search = search.trim();

                if(search === '') {
                    this.displayed = this.selected;
                    this.loading = false;
                    return;
                }

                this.loading = true;

                this.ajaxes.forEach(t => {
                    clearTimeout(t)
                });

                let vm = this;
                let srch = search;

                let t = setTimeout(function() {
                    let baseUrl = '/nova-vendor/nova-attach-many/';

                    Nova.request(baseUrl + vm.resourceName + '/attachable/' + vm.field.attribute + '?search=' + srch)
                        .then((data) => {
                            vm.available = data.data.available || [];
                            vm.displayed = vm.selected.concat(vm.available);
                            vm.loading = false;

                            vm.displayed = vm.displayed.reduce((unique, o) => {
                                if(!unique.some(obj => obj.label === o.label && obj.value === o.value)) {
                                    unique.push(o);
                                }
                                return unique;
                            },[]);
                        });

                    vm.checkIfSelectAllIsActive();
                }, 500);

                this.ajaxes.push(t);
            }
        },
        'selected': {
            handler: function (selected) {
                selected = selected.map(a => a.value);
                this.value = JSON.stringify(selected);
                this.checkIfSelectAllIsActive();
            },
            deep: true
        }
    }
}
</script>
