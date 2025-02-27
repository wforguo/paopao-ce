<template>
    <div>
        <main-nav title="主页" />

        <n-list
            class="main-content-wrap profile-wrap"
            bordered
            v-if="store.state.userInfo.id > 0"
        >
            <!-- 基础信息 -->
            <div class="profile-baseinfo">
                <div class="avatar">
                    <n-avatar :size="72" :src="store.state.userInfo.avatar" />
                </div>
                <div class="base-info">
                    <div class="username">
                        <strong>{{ store.state.userInfo.nickname }}</strong>
                        <span> @{{ store.state.userInfo.username }} </span>
                        <n-tag v-if="store.state.userInfo.is_admin" class="top-tag" type="error" size="small" round>
                            管理员
                        </n-tag>
                    </div>
                    <div class="userinfo">
                        <span class="info-item">UID. {{ store.state.userInfo.id }} </span>
                        <span class="info-item">{{ formatDate(store.state.userInfo.created_on) }}&nbsp;加入</span>
                    </div>
                    <div class="userinfo">
                        <span class="info-item">
                             <router-link
                                @click.stop
                                class="following-link"
                                :to="{
                                    name: 'following',
                                    query: { 
                                        s: store.state.userInfo.username, 
                                        n: store.state.userInfo.nickname,
                                        t: 'follows',
                                    },
                                }"
                            >
                                关注&nbsp;&nbsp;{{ store.state.userInfo.follows }}
                            </router-link>
                        </span>
                        <span class="info-item">
                            <router-link
                                @click.stop
                                class="following-link"
                                :to="{
                                    name: 'following',
                                    query: { 
                                        s: store.state.userInfo.username, 
                                        n: store.state.userInfo.nickname,
                                        t: 'followings',
                                    },
                                }"
                            >
                                粉丝&nbsp;&nbsp;{{ store.state.userInfo.followings }}
                            </router-link>
                        </span>
                    </div>
                </div>
            </div>
            <n-tabs class="profile-tabs-wrap" type="line" animated @update:value="changeTab">
                <n-tab-pane name="post" tab="泡泡"> </n-tab-pane>
                <n-tab-pane name="comment" tab="评论"> </n-tab-pane>
                <n-tab-pane name="highlight" tab="亮点"> </n-tab-pane>
                <n-tab-pane name="media" tab="图文"> </n-tab-pane>
                <n-tab-pane name="star" tab="喜欢"> </n-tab-pane>
            </n-tabs>
            <div v-if="loading" class="skeleton-wrap">
                <post-skeleton :num="pageSize" />
            </div>
            <div v-else>
                <div class="empty-wrap" v-if="list.length === 0">
                    <n-empty size="large" description="暂无数据" />
                </div>

                <div v-if="store.state.desktopModelShow">
                    <n-list-item v-for="post in list" :key="post.id">
                        <post-item :post="post" />
                    </n-list-item>
                </div>
                <div v-else>
                    <n-list-item v-for="post in list" :key="post.id">
                        <mobile-post-item :post="post" />
                    </n-list-item>
                </div>
            </div>
        </n-list>

        <n-space v-if="totalPage > 0" justify="center">
            <InfiniteLoading class="load-more" :slots="{ complete: '没有更多泡泡了', error: '加载出错' }" @infinite="nextPage()">
                <template #spinner>
                    <div class="load-more-wrap">
                        <n-spin :size="14" v-if="!noMore" />
                        <span class="load-more-spinner">{{ noMore ? '没有更多泡泡了' : '加载更多' }}</span>
                    </div>
                </template>
            </InfiniteLoading>
        </n-space>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { useStore } from 'vuex';
import { useRoute } from 'vue-router';
import { getUserPosts } from '@/api/user';
import { formatDate } from '@/utils/formatTime';
import InfiniteLoading from "v3-infinite-loading";

const store = useStore();
const route = useRoute();
const loading = ref(false);
const noMore = ref(false);
const list = ref<Item.PostProps[]>([]);
const postList = ref<Item.PostProps[]>([]);
const commentList = ref<Item.PostProps[]>([]);
const highlightList = ref<Item.PostProps[]>([]);
const mediaList = ref<Item.PostProps[]>([]);
const starList = ref<Item.PostProps[]>([]);
const pageType = ref<"post" | "comment" | "highlight" |"media" | "star">('post');
const postPage = ref(+(route.query.p as string) || 1);
const commentPage = ref(1)
const highlightPage = ref(1)
const mediaPage = ref(1)
const starPage = ref(1);
const page = ref(+(route.query.p as string) || 1);
const pageSize = ref(20);
const totalPage = ref(0);
const postTotalPage = ref(0);
const commentTotalPage = ref(0);
const highlightTotalPage = ref(0);
const mediaTotalPage = ref(0);
const starTotalPage = ref(0);

const loadPage = () => {
    switch(pageType.value) {
        case "post":
            loadPosts();
            break;
        case "comment":
            loadCommentPosts();
            break;
        case "highlight":
            loadHighlightPosts();
            break;
        case "media":
            loadMediaPosts();
            break;
        case "star":
            loadStarPosts();
            break;
    } 
};
const loadPosts = () => {
    loading.value = true;
    getUserPosts({
        username: store.state.userInfo.username,
        style: "post",
        page: page.value,
        page_size: pageSize.value,
    })
        .then((rsp) => {
            loading.value = false;
            if (rsp.list.length === 0) {
                noMore.value = true
            }
            if (page.value > 1) {
                list.value = list.value.concat(rsp.list);
            } else {
                list.value = rsp.list || [];
                window.scrollTo(0, 0);
            }
            totalPage.value = Math.ceil(rsp.pager.total_rows / pageSize.value);
            postList.value = list.value;
            postTotalPage.value = totalPage.value;
        })
        .catch((err) => {
            list.value = [];
            if (page.value > 1) {
                page.value--;
            }
            loading.value = false;
        });
};
const loadCommentPosts = () => {
    loading.value = true;
    getUserPosts({
        username: store.state.userInfo.username,
        style: "comment",
        page: page.value,
        page_size: pageSize.value,
    })
        .then((rsp) => {
            loading.value = false;
            if (rsp.list.length === 0) {
                noMore.value = true
            }
            if (page.value > 1) {
                list.value = list.value.concat(rsp.list);
            } else {
                list.value = rsp.list || [];
                window.scrollTo(0, 0);
            }
            totalPage.value = Math.ceil(rsp.pager.total_rows / pageSize.value);
            commentList.value = list.value;
            commentTotalPage.value = totalPage.value;
        })
        .catch((err) => {
            list.value = [];
            if (page.value > 1) {
                page.value--;
            }
            loading.value = false;
        });
};
const loadHighlightPosts = () => {
    loading.value = true;
    getUserPosts({
        username: store.state.userInfo.username,
        style: "highlight",
        page: page.value,
        page_size: pageSize.value,
    })
        .then((rsp) => {
            loading.value = false;
            if (rsp.list.length === 0) {
                noMore.value = true
            }
            if (page.value > 1) {
                list.value = list.value.concat(rsp.list);
            } else {
                list.value = rsp.list || [];
                window.scrollTo(0, 0);
            }
            totalPage.value = Math.ceil(rsp.pager.total_rows / pageSize.value);
            highlightList.value = list.value;
            highlightTotalPage.value = totalPage.value;
        })
        .catch((err) => {
            list.value = [];
            if (page.value > 1) {
                page.value--;
            }
            loading.value = false;
        });
};
const loadMediaPosts = () => {
    loading.value = true;
    getUserPosts({
        username: store.state.userInfo.username,
        style: "media",
        page: page.value,
        page_size: pageSize.value,
    })
        .then((rsp) => {
            loading.value = false;
            if (rsp.list.length === 0) {
                noMore.value = true
            }
            if (page.value > 1) {
                list.value = list.value.concat(rsp.list);
            } else {
                list.value = rsp.list || [];
                window.scrollTo(0, 0);
            }
            totalPage.value = Math.ceil(rsp.pager.total_rows / pageSize.value);
            mediaList.value = list.value;
            mediaTotalPage.value = totalPage.value;
        })
        .catch((err) => {
            list.value = [];
            if (page.value > 1) {
                page.value--;
            }
            loading.value = false;
        });
};
const loadStarPosts = () => {
    loading.value = true;
    getUserPosts({
        username: store.state.userInfo.username,
        style: "star",
        page: page.value,
        page_size: pageSize.value,
    })
        .then((rsp) => {
            loading.value = false;
            if (rsp.list.length === 0) {
                noMore.value = true
            }
            if (page.value > 1) {
                list.value = list.value.concat(rsp.list);
            } else {
                list.value = rsp.list || [];
                window.scrollTo(0, 0);
            }
            totalPage.value = Math.ceil(rsp.pager.total_rows / pageSize.value);
            starList.value = list.value;
            starTotalPage.value = totalPage.value;
        })
        .catch((err) => {
            list.value = [];
            if (page.value > 1) {
                page.value--;
            }
            loading.value = false;
        });
};
const changeTab = (tab: "post" | "comment" | "highlight" | "media" | "star") => {
    pageType.value = tab;
    switch(pageType.value) {
        case "post":
            list.value = postList.value;
            page.value = postPage.value;
            totalPage.value = postTotalPage.value;
            loadPosts();
            break;
        case "comment":
            list.value = commentList.value;
            page.value = commentPage.value;
            totalPage.value = commentTotalPage.value;
            loadCommentPosts();
            break;
        case "highlight":
            list.value = highlightList.value;
            page.value = highlightPage.value;
            totalPage.value = highlightTotalPage.value;
            loadHighlightPosts();
            break;
        case "media":
            list.value = mediaList.value;
            page.value = mediaPage.value;
            totalPage.value = mediaTotalPage.value;
            loadMediaPosts();
            break;
        case "star":
            list.value = starList.value;
            page.value = starPage.value;
            totalPage.value = starTotalPage.value;
            loadStarPosts();
            break;
    }
};
const updatePage = () => {
    switch(pageType.value) {
        case "post":
            postPage.value = page.value;
            loadPosts();
            break;
        case "comment":
            commentPage.value = page.value;
            loadCommentPosts();
            break;
        case "highlight":
            highlightPage.value = page.value;
            loadHighlightPosts();
            break;
        case "media":
            mediaPage.value = page.value;
            loadMediaPosts();
            break;
        case "star":
            starPage.value = page.value;
            loadStarPosts();
            break;
    }
};
const nextPage = () => {
    if (page.value < totalPage.value || totalPage.value == 0) {
        noMore.value = false;
        page.value++;
        updatePage();
    }  else {
        noMore.value = true;
    }
};
onMounted(() => {
    loadPage();
});
watch(
    () => ({
        path: route.path,
        query: route.query,
        refresh: store.state.refresh,
    }),
    (to, from) => {
        if (to.refresh !== from.refresh) {
            page.value = +(route.query.p as string) || 1;
            setTimeout(() => {
                loadPage();
            }, 0);
            return;
        }
        if (from.path !== '/post' && to.path === '/profile') {
            page.value = +(route.query.p as string) || 1;
            setTimeout(() => {
                loadPage();
            }, 0);
        }
    }
);
</script>

<style lang="less" scoped>
.profile-baseinfo {
    display: flex;
    padding: 16px;
    .avatar {
        width: 72px;
    }

    .base-info {
        position: relative;
        margin-left: 12px;
        width: calc(100% - 84px);

        .username {
            line-height: 16px;
            font-size: 16px;
        }

        .userinfo {
            font-size: 14px;
            line-height: 14px;
            margin-top: 10px;
            opacity: 0.75;
            .info-item {
                margin-right: 12px;
            }
        }

        .top-tag {
            transform: scale(0.75);
        }
    }
}

.profile-tabs-wrap {
    padding: 0 16px;
}

.load-more {
    margin: 20px;

    .load-more-wrap {
        display: flex;
        flex-direction: row;
        justify-content: center;
        align-items: center;
        gap: 14px;

        .load-more-spinner {
            font-size: 14px;
            opacity: 0.65;
        }
    }
}

.dark {
    .profile-wrap, .pagination-wrap {
        background-color: rgba(16, 16, 20, 0.75);
    }
}
</style>