<template>
    <div class="conversations" :class="{conversations__wait: is_loading_conv}">
        <div v-show="is_loading" class="mercurius__loading">
            <svg class="ic"><use xlink:href="#icon-ani-puff"></use></svg>
            <h5 class="mt-3">{{ 'loading_conversations' | __ }}</h5>
        </div>

        <!-- New conversation container, ugly, but works for now -->
        <div class="conversation">
            <a
                href="#"
                v-show="is_appending"
                class="conversation active"
            >
                <mercurius-avatar
                    class="mr-3 pull-left"
                    size="60"
                    border="3"
                    name="New message"
                    :image="placeholder"
                ></mercurius-avatar>
                <h6 class="conversation__heading">
                    {{ 'new_message' | __ }}
                </h6>
                <p class="conversation__message">__</p>
            </a>
        </div>

        <!-- Append each conversation item -->
        <vue-scroll>
            <template v-for="conversation in filterData">
                <div
                    class="conversation"
                    :class="conversationClass(conversation)"
                    :key="conversation.slug"
                    >
                    <a
                        href="#"
                        @click.prevent="onOpen(conversation)"
                    >
                        <mercurius-avatar
                            class="mr-3 pull-left"
                            size="60"
                            :name="conversation.user"
                            :image="getAvatar(conversation)"
                            :is_online="conversation.is_online"
                        ></mercurius-avatar>
                        <h6 class="conversation__heading">
                            {{ conversation.user }}
                            <small class="conversation__date"
                            >{{ conversation.created_at | toDatetime}}</small>
                        </h6>
                        <div class="conversation__message">{{ getMsg(conversation) }}</div>
                    </a>

                    <!-- Delete Conversation -->
                    <div
                        class="conversation__action"
                        @click="deleteConversation(conversation.slug)"
                    ><svg class="ic ic-bin"><use xlink:href="#icon-bin"></use></svg>
                    </div>
                </div>
            </template>
        </vue-scroll>
    </div>
</template>

<script>
import ConversationsHttp from './conversations-http';

export default {
    props: ['conversations'],


    mixins: [
        ConversationsHttp,
    ],


    data() {
        return {
            active:          false,
            filterStr:       '',
            is_appending:    false,
            is_loading_conv: false,
            onLoadOpenFirst: false,
            placeholder:     '/vendor/mercurius/img/avatar/avatar_placeholder.png',
        }
    },


    created() {
        Bus.$on('mercuriusComposeNewMessage', () => this.onComposeNewMessage())
        Bus.$on('mercuriusConversationsLoaded', res => this.onConversationsLoaded(res));
        Bus.$on('mercuriusConversationLoaded', conv => this.onConversationLoaded(conv));
        Bus.$on('mercuriusFilterConversations', str => this.onFilter(str))
        Bus.$on('mercuriusMessageDeleted', (msg, msgLatest) => this.onMessageDeleted(msg, msgLatest))
        Bus.$on('mercuriusMessageReceived', (usr, sender, msg) => this.onMessageReceived(sender, msg))
        Bus.$on('mercuriusMessageSent', msg => this.refreshMessage(msg))
        Bus.$on('mercuriusRecipientSelected', recipient => this.onRecipientSelected(recipient))
    },


    mounted() {
        this.loadConversations();
    },


    filters: {
        toDatetime(value) {
            let m = moment.utc(value).local()
            let h = moment.utc().local().diff(m, 'hours', true)

            if (h <= 24) return m.format('HH:mm')       // 14:00
            if (h <= 24*7) return m.format('ddd')       // Mon
            if (h <= 24*365) return m.format('D MMM')   // 25 Aug
            return m.format('DD/MM/YYYY')               // 25/08/2017
        }
    },


    computed: {
        sortedData() {
            return _.orderBy(this.conversations, 'created_at', 'desc');
        },
        filterData() {
            return this.sortedData.filter(c => {
                return c.user.toLowerCase()
                        .indexOf(this.filterStr.toLowerCase()) !== -1
            });
        },
    },


    methods: {
        conversationClass(conv) {
            return (!!this.active && this.active.slug === conv.slug ? 'active' : '')
                 + (this._received(conv) && _.isNull(conv.seen_at) ? ' unseen' : '');
        },
        getMsg(conv) {
            return (this._received(conv) ? '':'You: ') + conv.message
        },
        isTyping(conv) {
            return !_.isEmpty(conv.typing) && conv.typing
        },
        getAvatar(img) {
            return (img.avatar = img.avatar || this.placeholder)
        },
        refreshMessage(msg) {
            this.active.sender     = msg.sender
            this.active.message    = msg.message
            this.active.created_at = msg.created_at
        },


        // Private helpers
        //
        _find(slug) {
            return _.find(this.conversations, (c) => {
                return c.slug===slug
            })
        },
        _create(recipient) {
            let item = {
                "slug":       recipient.slug,
                "user":       recipient.name,
                "sender":     recipient.slug,
                "avatar":     recipient.avatar,
                "is_online":  recipient.is_online,
                "message":    '',
                "created_at": moment.utc().format('YYYY-MM-DD HH:mm:ss'),
            }
            this.conversations.unshift(item)

            return item
        },
        // Check if last message was received, or sent
        _received(conv) {
            return conv.sender!==Mercurius.user.slug
        },
        _findOrCreate(recipient) {
            return this._find(recipient.slug) || this._create(recipient)
        },


        // Event handlers
        //
        onOpen(conv) {
            if (this.is_loading_conv) return
            this.is_appending = false
            this.is_loading_conv = true
            this.active = conv
            Bus.$emit('mercuriusConversationOpen', conv);
        },
        onConversationLoaded(conv) {
            this.is_loading_conv = false
            this.active.seen_at = moment.utc().format('YYYY-MM-DD HH:mm:ss');
        },
        onConversationsLoaded(res) {
            if (this.onLoadOpenFirst && res.length > 0) this.onOpen(res[0])
        },
        onRecipientSelected(recipient) {
            let item = this._findOrCreate(recipient);
            this.onOpen(item)
        },
        onMessageReceived(sender, msg) {
            let item = this._findOrCreate(sender);
            item.sender  = sender.slug;
            item.message = msg.message;
            item.seen_at = null;
        },
        onMessageDeleted(msg, msgLatest) {
            if (!!msgLatest) this.refreshMessage(msgLatest)
        },
        onComposeNewMessage() {
            this.active = false
            this.is_appending = true
        },
        onFilter(str) {
            this.filterStr = str;
        },
    }
};
</script>
