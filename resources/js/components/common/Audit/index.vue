<template>
    <div class="audit">
        <el-col class="filter-col" v-if="showFilter">
            <el-divider :content-position="filterPosition">
                <el-popover
                    popper-class="popover-filter"
                    placement="bottom-end"
                    width="300"
                    v-model="visible"
                    trigger="click">
                        <svg-icon icon="filter" slot="reference" />
                        <history-filters ref="filters" :data="filters.data" :schema="filters.schema" @changed="filtersChanged" @update:data="filterReset" />
                        <!-- <el-button size="mini" icon="icon-eraser" @click="filterReset" type="success">{{$t('resident.reset_filters')}}</el-button> -->
                </el-popover>
            </el-divider>
        </el-col>
        <placeholder :src="require('img/5ce8f4e279cb2.png')" v-if="isEmpty">
            {{$t('resident.no_data.activity')}}
            <small>{{$t('resident.no_data_info.activity')}}</small>
        </placeholder>
        <el-timeline v-else class="custom-scroller max-height-300">
            <template v-for="audit in list">
                <el-timeline-item  :key="audit.id" :timestamp="`${formatDatetime(audit.updated_at)}`">
                    <avatar :src="audit.user.avatar" :name="audit.user.name" :size="32"/>
                    <span class="name">{{ audit.user.name }}</span>
                    <span>
                        {{audit.statement}}                            
                        <el-badge v-if="type == 'all'" :value="audit.auditable_format" class="item" type="primary"></el-badge>
                    </span>                                                
                </el-timeline-item>
            </template>
            <el-timeline-item v-if="loading">
                {{$t('resident.loading')}}
            </el-timeline-item>
            <div v-if="meta.current_page < meta.last_page" class="load-more">
                <el-button @click="loadMore" size="mini" style="margin-top: 15px" type="text">{{$t('general.load_more')}}</el-button>
            </div>
        </el-timeline>
    </div>
</template>

<script>
    import Placeholder from 'components/Placeholder'
    import {format} from 'date-fns'
    import queryString from 'query-string'
    import FormatDateTimeMixin from 'mixins/formatDateTimeMixin'
    import { EventBus } from '../../../event-bus.js';
    import auditFilter from './filters.json';
    import Avatar from 'components/Avatar';
    import HistoryFilters from 'components/HistoryFilters';

    export default {
        mixins: [FormatDateTimeMixin],

        props: {
            id: {
                type: [Number, String]
            },
            filterPosition: {
                default: 'right',
                type: String,
                validator: filterPosition => ['left', 'right'].includes(filterPosition)
            },
            showFilter: Boolean,
            type: {
                type: String,
                validator: type => ['pinboard', 'listing', 'request', 'unit', 'quarter', 'building', 'manager', 'resident', 'provider', 'all'].includes(type),                
            }
        },
        components: {
            Placeholder,
            Avatar,
            HistoryFilters,
        },
        data () {
            const filterSchema = [];

            const filterData = {
                event: null,
                auditable_type: null,
            };
            return {
                list: [],
                meta: {},
                filters: {
                    schema: filterSchema,
                    data: filterData
                },
                auditFilter: auditFilter,
                categories: [],
                loading: true,
                /*currentAuditableType: null,
                currentEvent: null,*/
                visible: false
            }
        },
        methods: {
            async filterReset(){
                let schema_children = [];
                let filter_name = '';
                this.filters = {
                    schema: [],
                    data: {
                        event: null,
                        auditable_type: null,
                    }
                };
                schema_children.push({
                    type: 'el-option',
                    props: {
                        label: 'resident.all',
                        value: null
                    }
                });
                /*if(this.type === 'all'){                      
                    // If there is no type prop on audit component then show type select
                    // Get filter translations from file
                    filter_name = 'auditable_type'                    
                    const filter_type_translations = this.$t(`general.components.common.audit.filter.type`);                    
                    const filter_type_options = Object.keys(filter_type_translations).map((key, index) => {                        
                    schema_children.push({
                        type: 'el-option',
                        props: {
                            label: filter_type_translations[key],
                            value: key
                            }
                        })
                    });                                       
                } else{                     */
                    // If there is type then only show event options
                    // Get type options from translation files
                    filter_name = 'event'
                    const filter_event_translations = this.auditFilter[this.type];                    
                    const filter_event_options = Object.values(filter_event_translations).map((value,key) => {                        
                        // Push to schema array
                        schema_children.push({
                            type: 'el-option',
                            props: {
                                label: `general.components.common.audit.filter.general.${value}`,
                                value: value
                            }
                        })
                    }); 
                // }
                this.filters.schema.push({
                    type: 'el-select',
                    title: 'resident.type',
                    name: filter_name,
                    props: {
                        size: 'mini'
                    },
                    children: schema_children
                })
            },
            async filtersChanged (filters) {                                                    
                // if((filters.auditable_type) && (filters.auditable_type != '') && (filters.auditable_type !== this.currentAuditableType)){                    
                if((filters.auditable_type) && (filters.auditable_type != '')){
                    let schema_children = [];
                    schema_children.push({
                        type: 'el-option',
                        props: {
                            label: 'resident.all',
                            value: null
                        }
                    });
                    const filter_event_translations = this.auditFilter[filters.auditable_type];
                    const filter_event_options = Object.values(filter_event_translations).map((value,key) => {                        
                        // Push to schema array
                        schema_children.push({
                            type: 'el-option',
                            props: {
                                label: `general.components.common.audit.filter.general.${value}`,
                                value: value
                            }
                        })
                    });                                             
                    //remove previous select if exists
                    this.filters.schema.splice(1,1)
                    //remove any set event data in filter
                    if(!Object.keys(filter_event_translations).includes(this.filters.data.event))
                    {
                        this.filters.data.event = null
                    }
                    this.filters.schema.push({
                        type: 'el-select',
                        title: 'Event type',
                        name: 'event',
                        props: {
                            size: 'mini'
                        },
                        children: schema_children
                    });
                    // this.currentAuditableType = filters.auditable_type;
                } 
                // else if((filters.event) && (filters.event !== this.currentEvent)){                    
                else if(filters.event){                    
                    // this.filters.schema.splice(1,1)                                    
                    if(this.filters.schema.findIndex(x => x.name == 'type') != -1){
                        this.filters.data.event = null
                    }
                    // this.currentEvent = filters.event;
                }
                this.list = undefined
                this.meta.current_page = undefined
                await this.fetch();
            },
            async fetch (page = 1,params) {
                // Get current page and last page of the displayed audits
                this.loading = true                

                const auditable_type = this.filters.data.auditable_type ? this.filters.data.auditable_type : this.type                
                // Fetch audits
                try {                    
                    const resp = await this.axios.get('audits?' + queryString.stringify({
                        sortedBy: 'desc',
                        orderBy: 'created_at',
                        page,
                        per_page: 25,
                        auditable_id: this.id,
                        auditable_type: auditable_type,
                        event: this.filters.data.event,
                        ...params,
                    }))  
                    this.meta = _.omit(resp.data.data, 'data');                  
                    if(!resp.data.data.data) {
                        this.list = []
                    }
                    else{                        
                        if (page === 1) {                            
                            this.list = resp.data.data.data;
                        } else {                            
                            this.list.push(...resp.data.data.data);
                        }
                        EventBus.$emit('audit-get-counted', resp.data.data.total);                
                    }
                }catch (e) {
                    this.list = []
                    console.log(e);
                } finally {
                    this.loading = false;
                }                                             
            },
            loadMore() {
                if (this.meta.current_page < this.meta.last_page) {
                    this.fetch(this.meta.current_page + 1);
                }                
            }
        },
        computed: {
            isEmpty () {
                return !this.loading && !Object.keys(this.list).length
            }
        },
        watch: {
            '$i18n.locale' : function(val) {                
                this.fetch();
            },
        },
        async mounted () {               
            // const {data:{data}} = await this.axios.get('requestCategories/tree?get_all=true');
            // // Get filter options from translation file and add the to filter object

            // const flattenCategories = categories => categories.reduce((obj, category) => {
            //     obj[category.id] = category.name_en.toLowerCase().replace(/ /g,"_");

            //     if (category.categories) {
            //         obj = {...obj, ...flattenCategories(category.categories)}

            //         delete category.categories;
            //     }
            //     return obj
            // }, {})

            // this.categories = flattenCategories(data)
            this.categories = this.$constants.requests.categories_data.tree
            // if(this.showFilter){
            //     await this.filterReset();            
            // }                 
            await this.filterReset();       
            await this.fetch();            
        }
    }
</script>

<style lang="scss" scoped>
    .audit {
        height: 100%;
        display: flex;
        flex-direction: column;
        .el-badge {
            display: inline-block;
            margin: 2px 0 0 5px;
        }
        .placeholder {
            font-size: 16px;
            small {
                color: darken(#fff, 28%);
            }
        }
        .el-divider__text {
            display: flex;
            align-items: center;
            i {
                margin-right: 10px;
            }
        }
        .el-divider__text.is-left{
            left: 0;
            padding:0;
        }
        .el-divider__text.is-right{
            right: 0;
            padding:0;
        }
        /deep/ .el-popover__reference {
            padding-left: 5px;
            svg {
                font-size: 15px;
                
                &:hover {
                    cursor: pointer;
                }
            }
        }
        .filter-col {
            padding-left: 0 !important;
            padding-right: 0 !important;

            .el-divider {
                background-color: transparent;
                margin: 5px 0px 15px 0px;
            }

        }
        .el-timeline {
            padding: 0 0 0 1px;
            height: 100%;
            overflow-y: auto;
            overflow-x: hidden;

            &.max-height-300 {
                max-height: 300px;
            }
            
            .audit-timestamp{
                font-size:11px;
                color:#9e9e9e;
            }
            .el-timeline-item {
                &:last-of-type {
                    padding-bottom: 0px;
                }
                
                :global(.el-timeline-item__tail) {
                    top: 13px;
                    left: 16.5px;
                    display: none;
                }
                :global(.el-timeline-item__node) {
                    top: 13px;
                    left: 15px;
                }
                :global(.el-timeline-item__wrapper) {
                    position: relative;
                    padding-left: 45px;
                    padding-right: 10px;
                    .el-timeline-item__content {
                        color: #9E9FA0;
                        span {
                            color: #9E9FA0;
                        }
                        span:first-of-type {
                            font-weight: 900;
                            color: #3d3f41;
                        }
                        .avatar {
                            position: absolute;
                            left: 0px;
                            top: 5px;
                            border: 1px solid  var(--border-color-base);
                        }
                    }
                 }
            } 
            .load-more .el-button {
                background: transparent;
                color: #3d3f41;
                font-size: 14px;
                cursor: pointer;
            }
        }

}
</style>
<style lang="scss">
.popover-filter {
    padding: 10px 0;
    border: 1px solid #707070;
    box-shadow: 0 0px 3px 0 rgba(0,0,0,.1);

    .popper__arrow {
        border-bottom-color: #707070 !important;
    }

    .el-button [class*="icon-"] + span{
        margin-left: 5px;
    }
    .el-button{
        width: 100%;
        margin-top: 10px;
    }
}
</style>