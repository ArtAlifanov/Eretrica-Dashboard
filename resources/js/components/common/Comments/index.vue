<template>
    <div class="placeholder" v-if="!loading && !comments.data.length">
        <template v-if="usePlaceholder">
            <img class="image" :src="require('img/5c98b47a97050.png')" />            
                <div class="title">{{$t(no_data_info.title)}}</div>
                <div class="description">{{$t(no_data_info.description)}}</div>       
        </template>
    </div>
    <div class="comments-list" v-else-if="comments.data.length">
        <template v-if="withScroller" >
            <dynamic-scroller ref="dynamic-scroller" class="custom-scroller" :items="comments.data" :min-item-size="60" v-if="!loading">
                <template #before v-resize:debounce="onResize">
                    <el-divider v-if="comments.current_page !== comments.last_page">
                        <el-button icon="el-icon-top" size="mini" :loading="loading" round @click="fetch">
                            <template v-if="loading">{{$t('general.loading')}}</template>
                            <template v-else>{{$t('general.load_more_count', {count: comments.total - comments.data.length})}}</template>
                        </el-button>
                    </el-divider>
                </template>
                <template v-slot="{item, index, active}">
                    <dynamic-scroller-item :item="item" :active="active" :data-index="index" :size-dependencies="[item]">
                        <div v-if="item.date_label && newStyle" class="chat-date">
                            <span v-resize:debounce="onResize">{{ item.date_label }}</span>
                        </div>
                        <comment
                            :newStyle="newStyle"
                            :type="type"
                            v-bind="commentComponentProps" 
                            v-on="commentComponentListeners" 
                            :show-children="showChildren" 
                            :data="item" 
                            :reversed="isCommentReversed(item)"
                            v-resize:debounce="onResize" />

                        <!-- <internal-notices-comment
                            v-else 
                            v-bind="commentComponentProps" 
                            v-on="commentComponentListeners" 
                            :show-children="showChildren" 
                            :data="item" 
                            :reversed="isCommentReversed(item)"
                            v-resize:debounce="onResize" /> -->
                    </dynamic-scroller-item>
                </template>
            </dynamic-scroller>
        </template>
        <template v-else>
            <el-button type="text" @click="fetch" :loading="loading" v-if="!data && comments.current_page !== comments.last_page">
                {{$t('general.components.common.comments_list.load_more', {count: comments.total - comments.data.length})}}
            </el-button>            
            <comment v-bind="commentComponentProps" :show-children="showChildren" v-for="comment in comments.data" :key="comment.id" :data="comment" :reversed="isCommentReversed(comment)" />
        </template>
    </div>
</template>

<script>
    import { format, addMinutes, differenceInCalendarDays } from 'date-fns'
    import Loader from 'components/common/Comments/Loader'
    import {displaySuccess, displayError} from 'helpers/messages'
    import { EventBus } from '../../../event-bus.js';
    import resize from 'vue-resize-directive';
    import InternalNoticesComment from 'components/InternalNoticesComment';

    export default {
        props: {
            id: {
                type: Number,
            },
            parentId: {
                type: Number
            },
            type: {
                type: String,
                validator: type => ['pinboard', 'listing', 'request', 'conversation', 'internalNotices'].includes(type)
            },
            data: {
                type: Object
            },
            limit: {
                type: Number,
                default: 5
            },
            reversed: {
                type: Boolean,
                default: false
            },
            withScroller: {
                type: Boolean,
                default: false
            },
            showChildren: {
                type: Boolean,
                default: true
            },
            usePlaceholder: {
                type: Boolean,
                default: true
            },            
            newStyle: {
                type: Boolean,
                default: false,
            }
        },
        directives: {
            resize
        },
        components: {
            Loader,
            InternalNoticesComment,
        },
        data () {
            return {
                loader: null,
                loading: true,
                comments: {
                    data: []
                },
                isShow: [],
            }
        },
        methods: {
            async fetch (params) {
                const {
                    current_page,
                    last_page
                } = this.comments;

                if (current_page && last_page &&
                    current_page == last_page) {
                    return
                }

                let prevScrollHeight = 0

                if (this.$refs['dynamic-scroller']) {
                    prevScrollHeight = this.$refs['dynamic-scroller'].$el.scrollHeight
                }


                let page = current_page || 0;

                page++;

                this.loading = true

                try {
                    await this.$store.dispatch('comments/get', {
                        id: this.id,
                        page,
                        per_page: this.limit,
                        sortedBy: 'asc',
                        orderBy: 'created_at',
                        commentable: this.type,
                        request_id: this.type == 'internalNotices' || this.type == 'conversation' ? this.id : '',
                        ...params
                    })

                    this.comments = this.$store.getters['comments/get'](this.id, this.type)
                    if(this.type === 'internalNotices'){
                        EventBus.$emit('notice-comment-count', this.comments.total);
                    } else if(this.type === 'pinboard'){
                        EventBus.$emit('pinboard-comment-count', this.comments.total);
                    } else if(this.type === 'request'){
                        EventBus.$emit('request-comment-count', this.comments.total);
                    }                    

                    if (this.$refs['dynamic-scroller'] && current_page >= 1) {
                        //this.$refs['dynamic-scroller'].$el.scrollTop = this.$refs['dynamic-scroller'].$el.scrollHeight - prevScrollHeight
                    }
                } catch (err) {
                    displayError(err)
                } finally {
                    this.loading = false
                }
            },
            forceUpdate () {
                this.$refs['dynamic-scroller'].forceUpdate()
            },
            scrollToBottom () {
                if (this.$refs['dynamic-scroller']) {
                    this.$refs['dynamic-scroller'].scrollToBottom()
                }
            },
            refreshComments () {
                this.comments = {
                    data: []
                }

                this.fetch()
            },
            isCommentReversed (comment) {
                return this.reversed && comment.user_id !== this.$store.getters.loggedInUser.id
            },
            onResize() {
                this.$emit('update-dynamic-scroller');
                //this.scrollToBottom();
            },
            getDateLabel(date) {
                let now = new Date();
                let server_now = addMinutes(now, now.getTimezoneOffset() + 60);
                let diff = differenceInCalendarDays(server_now, date);
                let result = '';
                if(diff === 0)
                    result = this.$t('general.today');
                else if(diff === 1)
                    result = this.$t(`general.yesterday`);
                else if(diff && diff < 7) {
                    result = this.$t(`general.days.${format(new Date(date), 'dddd').toLowerCase()}`);
                } else if(diff) {
                    result = this.$t(`general.days.${format(new Date(date), 'dddd').toLowerCase()}`) + format(new Date(date), ', DD.MM.YYYY');
                }
                if(!this.isShow[date.slice(0, 11)])
                    this.isShow[date.slice(0, 11)] = result;
                return result;
            },
            initDateLabel() {
                this.isShow = [];
                this.comments.data.forEach((item) => {
                    if(!this.isShow[item.created_at.slice(0, 11)])
                        item.date_label = this.getDateLabel(item.created_at)
                    else
                        item.date_label = '';
                });
                
            }
        },
        computed: {
            hasEmptyComments () {
                return !this.loading && !this.comments.data.length
            },
            commentComponentProps() {
                let props = {};

                if (this.id) {
                    props.id = this.id
                }

                if (this.parentId) {
                    props.parentId = this.parentId
                }

                if (this.type) {
                    props.type = this.type
                }

                return props
            },
            commentComponentListeners () {
                return {
                    'enter-edit': this.forceUpdate,
                    'cancel-edit': this.forceUpdate,
                    'size-changed': this.forceUpdate
                }
            },
            no_data_info(){
                let macros = {
                    title: 'general.components.common.comments_list.empty_placeholder.title',
                    description: 'general.components.common.comments_list.empty_placeholder.description'
                }
                if(this.type === 'internalNotices'){                    
                    macros.title = 'general.components.common.internalnotices_list.empty_placeholder.title';
                    macros.description = 'general.components.common.internalnotices_list.empty_placeholder.description';                    
                }
                else if(this.type === 'conversation'){                    
                    macros.title = 'general.components.common.serviceproviderconversations_list.empty_placeholder.title';
                    macros.description = 'general.components.common.serviceproviderconversations_list.empty_placeholder.description';                    
                }
                else if((this.type === 'request') && (this.$store.getters.loggedInUser.roles.findIndex(({name}) => name === 'resident') == -1)){ 
                    macros.title = 'general.components.common.residentconversations_list.empty_placeholder.title';
                    macros.description = 'general.components.common.residentconversations_list.empty_placeholder.description';                    
                }
                else if((this.type === 'listing') && (this.$store.getters.loggedInUser.roles.findIndex(({name}) => name === 'resident') > -1)){
                    macros.title = 'general.components.common.listingcomments_list.empty_placeholder.title';
                    macros.description = 'general.components.common.listingcomments_list.empty_placeholder.description';                    
                }
                else if((this.type === 'pinboard')){
                    macros.title = 'general.components.common.pinboardcomments_list.empty_placeholder.title';
                    macros.description = 'general.components.common.pinboardcomments_list.empty_placeholder.description';
                }
                return macros;
            } 
        },
        watch: {
            'comments.total' () {
                this.scrollToBottom();
                this.initDateLabel();
            },
            'data' (comments) {
                this.comments = comments;
                this.initDateLabel();
            }
        },
        async mounted() {
            if (this.data) {
                this.comments = this.data;
            } else {
                
                this.$store.dispatch('comments/clear', {commentable: this.type})

                await this.fetch()
            }

            this.comments.data.reverse();
            this.initDateLabel();
            this.$emit('update-dynamic-scroller')
        },
        created() {
            
        }
    }
</script>

<style lang="sass" scoped>
    .placeholder
        display: flex
        padding: 16px
        text-align: center
        align-items: center
        flex-direction: column
        justify-content: center

        .image
            width: 128px

        .title
            font-size: 17px
            font-weight: 800
            color: var(--color-primary)

        .description
            font-size: 14px
            font-weight: 600
            word-break: break-word
            color: var(--color-text-placeholder)

    .comments-list
        width: 100%
        display: flex
        position: relative
        flex-direction: column
        box-sizing: border-box
        min-height: 300px

        svg
            &:not(:last-of-type)
                margin-bottom: 4px

        .vue-recycle-scroller
            height: 100%
            min-height: 300px
            overflow: auto
            will-change: transform

            /deep/ .vue-recycle-scroller__item-wrapper
                min-height: (inherit !important)
                overflow: visible

        .el-button
            align-self: flex-start

        .chat-date
            text-align: center;
            height: 50px
            display: flex
            justify-content: center
            align-items: flex-start
            span 
                background-color: #F0F0F0;
                border-radius: 12px
                color: #9E9FA0
                padding: 3px 15px;
                margin-bottom: 10px
</style>