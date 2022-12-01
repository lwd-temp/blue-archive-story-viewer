<script setup lang="ts">
import { PropType, Ref, ref } from 'vue';
import { CurrentMessageItem, Momotalk } from '../../types/Chats';
import MomoTalkComponent from './MomoTalkComponent.vue';

const props = defineProps({
  messageGroup: Number,
  translate: {
    type: String,
    default: '',
    required: false,
  },
  content: Object as PropType<Momotalk[]>,
});

const messageList: Ref<CurrentMessageItem[]> = ref([]);
const pushedMessages: Ref<number[]> = ref([]);

const firstItem = props.content?.[0] as CurrentMessageItem;
firstItem.avatar = true; // 第一条消息显示学生头像
messageList.value.push(firstItem as CurrentMessageItem);

function messageToTypeofSelection(message: Momotalk[]): CurrentMessageItem {
  const mergedMessage = message[0] as CurrentMessageItem;
  mergedMessage.avatar = false;
  if (1 === message.length) {
    const messageElement = message[0] as Momotalk;
    mergedMessage.options = [
      {
        MessageKR: messageElement.MessageKR,
        MessageJP: messageElement.MessageJP,
        MessageCN: messageElement.MessageCN,
        MessageEN: messageElement.MessageEN,
        MessageTH: messageElement.MessageTH,
        MessageTW: messageElement.MessageTW,
        NextGroupId: messageElement.NextGroupId,
      },
    ];
  } else {
    mergedMessage.options = [];
    for (const item of message) {
      mergedMessage.options.push({
        MessageKR: item.MessageKR,
        MessageJP: item.MessageJP,
        MessageCN: item.MessageCN,
        MessageEN: item.MessageEN,
        MessageTH: item.MessageTH,
        MessageTW: item.MessageTW,
        NextGroupId: item.NextGroupId,
      });
    }
  }
  return mergedMessage;
}

function getMessagesByNextGroupId(nextGroupId: number): CurrentMessageItem[] {
  const nextMessages = props.content?.filter(
    message => nextGroupId === message.MessageGroupId
  ) as CurrentMessageItem[];
  // 消息会返回一个消息组，分三种情况讨论：
  // 1. 消息组只有一条消息，检测一下类型是不是 Answer
  // 2. 消息组有多条消息，但是没有 MessageCondition === Answer 的消息，全部返回
  // 3. 消息组有多条消息，并且是玩家选项，合并返回
  // 第一种情况
  if (1 === nextMessages.length) {
    return 'Answer' === nextMessages[0].MessageCondition
      ? [messageToTypeofSelection(nextMessages)]
      : nextMessages;
  }
  // 第二、第三种情况
  const answerMessages = nextMessages.filter(
    message => 'Answer' === message.MessageCondition
  );
  if (0 === answerMessages.length) {
    return nextMessages;
  } else {
    return [messageToTypeofSelection(answerMessages)];
  }
}

// 判断要不要显示学生头像
function shouldShowAvatar(
  previousCondition: 'None' | 'FavorRankUp' | 'Answer' | 'Feedback',
  currentCondition: 'None' | 'FavorRankUp' | 'Answer' | 'Feedback'
): boolean {
  if (previousCondition === currentCondition) {
    return false;
  } else {
    return 'Answer' === previousCondition;
  }
}

// 处理下一条消息事件
function handleNextMessage(Id: number, nextGroupId: number) {
  const previousMessage = messageList.value.find(
    message => Id === message.Id
  ) as CurrentMessageItem;
  const nextMessages = getMessagesByNextGroupId(nextGroupId);

  if (0 === nextMessages.length) {
    // 没有下一条消息，结束
    return;
  }
  if (1 === nextMessages.length) {
    // 只有一条消息，直接推入
    const nextMessage = nextMessages[0] as CurrentMessageItem;
    nextMessage.avatar = shouldShowAvatar(
      previousMessage.MessageCondition,
      nextMessage.MessageCondition
    );
    messageList.value.push(nextMessage);
    pushedMessages.value.push(nextMessage.Id);
  } else {
    // 有多条消息，选择第一条不在已经推送列表中的消息进行推送
    const nextMessage = nextMessages.filter(message => message.Id > Id)[0];
    messageList.value.push(nextMessage);
  }
}

// 处理用户选择和重新选择事件
function handleUserSelect(Id: number, nextGroupId: number) {
  const selectionIndex = messageList.value.findIndex(value => value.Id === Id);
  messageList.value = messageList.value.slice(0, selectionIndex + 1);
  pushedMessages.value = pushedMessages.value.slice(0, selectionIndex + 1);
  handleNextMessage(Id, nextGroupId);
}
</script>

<template>
  <div class="momotalk-main-interface flex-vertical">
    <div class="momotalk-banner flex-horizontal center">
      <!-- eslint-disable max-len -->
      <svg
        xmlns="http://www.w3.org/2000/svg"
        xml:space="preserve"
        id="Layer_2_00000033347787382183846980000016369978894466167968_"
        x="0"
        y="0"
        viewBox="0 0 512 477.9"
      >
        <path
          d="M101.8 396.4c-53.3 0-98.4 53.8-101.8 79.5 81.4 7.9 196.1-6.8 252.1-65-74.7 13.5-135.9-5.2-150.3-14.5zM281.4 412.5c55.1 73.3 137.7 58.5 230.5 63.3 3.5-42.1-73.2-80.9-103.9-82.2-39.9 28.5-123.3 19.5-126.6 18.9z"
        />
        <path
          d="M256.4 0C136.8 45 31.5 151.4 31.5 259.4c0 85 68.4 153.9 195.3 137.3 3.9-.5 7.6-1.1 11.2-1.7-1.1-.4-2.1-.9-2.9-1.3-19.4-9.8-53.4-56.7-39-112.6 1.4 59.3 39.7 102.6 57.1 111 2.7 1.3 11.9 4.6 25.5 6.7 51.9 8 164.9 4.7 187.9-116.7C498.9 111.4 256.4 0 256.4 0z"
        />
      </svg>
      <!-- eslint-enable max-len -->
      <div class="momotalk-title-text">
        <span class="title">MomoTalk</span>
        <div class="credit">
          <span>translated by </span>
          <span v-if="translate">{{ translate }}@</span>
          <a href="https://space.bilibili.com/37507923" target="_blank">
            碧蓝档案资讯站
          </a>
        </div>
      </div>
    </div>
    <div class="messages">
      <momo-talk-component
        v-for="(message, index) in messageList"
        :key="index"
        :message="message"
        @userSelect="handleUserSelect"
        @resolveNextMessage="handleNextMessage"
      />
    </div>
  </div>
</template>

<style scoped lang="scss">
.momotalk-main-interface {
  margin-top: 1rem;
  padding-bottom: 2rem;
  width: 100%;
}

.messages {
  width: 100%;
}

.momotalk-banner {
  background-color: var(--color-momotalk-background);
  padding: 0.5rem;
  width: 100%;
  color: var(--color-momotalk-banner-text);
  user-select: none;

  svg {
    fill: var(--color-momotalk-banner-text);
    width: 2rem;
    height: 2rem;
  }

  .momotalk-title-text {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    margin-left: 0.5rem;
    font-size: 1.25rem;
    font-family: 'Asap Condensed', 'Microsoft YaHei', 'PingFang SC',
      -apple-system, system-ui, 'Segoe UI', Roboto, Ubuntu, Cantarell,
      'Noto Sans', BlinkMacSystemFont, 'Helvetica Neue', 'Hiragino Sans GB',
      Arial, sans-serif;

    .title {
      letter-spacing: 0.05rem;
    }

    .credit {
      font-style: italic;
      font-size: 0.5rem;

      a {
        display: inline-flex;
        color: var(--color-momotalk-banner-text);
      }
    }
  }

  .messages {
    width: 100%;
  }
}
</style>
