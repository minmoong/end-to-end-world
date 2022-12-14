<script lang="ts">
  import { onMount } from 'svelte';
  import { usedWords, myCounter } from 'src/store';
  import { getNewWord, getStartWord, addScore } from 'src/functions';
  import Ripple from '@smui/ripple';
  import Icons from './Icons.svelte';
  import Leaderboard from './Leaderboard.svelte';
  import { get } from 'svelte/store';
  import { pling_MP3 } from 'src/assets/audio';
  import { browser } from '$app/env';

  type message = {
    id: string;
    type: string;
    content: string;
    definition?: string;
    loading?: boolean;
  }

  let value = '';
  let messages: message[] = [];
  let messagesElement: HTMLDivElement;
  let sendWord: string = '';
  let isEnded = false;

  let receiveID = getRandomID();
  receiveMsg('', receiveID, undefined, true);

  onMount(async () => {
    const { startWord, definition } = await getStartWord();
    sendWord = startWord;
    changeMessages(true, receiveID, startWord, definition, false);
    usedWords.update(usedWords => usedWords.concat(startWord));
    
    if (browser) {
      window.addEventListener('resize', updateScroll);
    }
  });

  function updateScroll() {
    messagesElement?.scrollTo(0, messagesElement.scrollHeight);
  }

  function changeMessages(isReceive: boolean, id: string, content: string, definition?: string, loading?: boolean) {
    messages = messages.map(message => {
      if (message.id === id) {
        message.content = content;
        message.definition = definition;
        message.loading = loading;
      }
      return message;
    });

    if (isReceive) {
      let promise = (new Audio(pling_MP3)).play();
      if (promise !== undefined) {
        promise.then(_ => {
          // Autoplay started!
        }).catch(error => {
          // Autoplay was prevented.
          // Show a "Play" button so that user can start playback.
        });
      }
    }

    spreadScroll(500);
  }

  function getRandomID() {
    return (Math.random() + 1).toString(36).substring(7);
  }

  function sendMsg(msg: string, id: string) {
    messages = [...messages, { id, type: 'SEND', content: msg }];
  }

  function receiveMsg(msg: string, id: string, definition?: string, loading?: boolean) {
    messages = [...messages, { id, type: 'RECEIVE', content: msg, definition, loading }];
    spreadScroll(1000);
  }

  function spreadScroll(ms: number) {
    const ss = setInterval(updateScroll, 100);
    setTimeout(() => clearInterval(ss), ms);
  }

  async function onSubmit() {
    if (value === '') return;
    let word = value;
    value = '';

    let sendID = getRandomID();
    let receiveID = getRandomID();
    sendMsg(word, sendID);
    receiveMsg('', receiveID, undefined, true);

    const res = await getNewWord(sendWord, word, changeMessages, sendID);
    if (res.success) sendWord = res.message;
    if (res.end) {
      changeMessages(true, receiveID, (res.messages as string[])[0], undefined, false);
      receiveMsg((res.messages as string[])[1], 'END');
      addScore(1000);
      myCounter.set({ score: get(myCounter).score + 1000, increasement: 1000 });
      isEnded = true;
      return;
    }
    changeMessages(true, receiveID, res.message as string, res.definition, false);
    spreadScroll(500);
  }

</script>

<form on:submit|preventDefault={onSubmit} class="chat">
  <Leaderboard />
  <div class="messages" bind:this={messagesElement}>
    <div class="messages-content">
      <div class="message">
        <figure class="avatar">
          <Icons name="robot" class="avatar-icon" width="25px" height="25px" />
        </figure>
        ??? ???????????? ??????! ???
        <span></span>
      </div>
      {#each messages as { type, content, definition, loading }}
        {#if type === 'SEND'}
          <div class="message message-personal">
            {content}
            <div class="message-definition">
              {definition ? definition : ''}
            </div>
          </div>
          {:else if type === 'RECEIVE'}
          <div class="message" class:loading>
            <figure class="avatar">
              <Icons name="robot" class="avatar-icon" width="25px" height="25px" />
            </figure>
            <div class="message-text">
              {content}
            </div>
            <div class="message-definition">
              {definition ? definition : ''}
            </div>
            <span></span>
          </div>
        {/if}
      {/each}
    </div>
  </div>
  {#if !isEnded}
    <div class="message-box">
      <input
        type="text"
        placeholder="?????????..."
        autocomplete="off"
        bind:value
      />
      <span use:Ripple={{ surface: true }}>
        <button>
          {#if value === ''}
            <Icons name="send-outlined" width="23" height="23" />
            {:else}
              <Icons name="send-filled" width="23" height="23" />
          {/if}
        </button>
      </span>
    </div>
    {:else}
      <div class="restart" on:click={() => { location.reload(); }}>
        <div class="restart-name">?????? ??????</div>
        <Icons name="refresh" width="20" height="20" class="refresh-icon" />
      </div>
  {/if}
</form>

<style lang="scss">
  .chat {
    @include desktop {
      position: absolute;
      left: 50%;
      transform: translateX(-50%);
      width: 100%;
      height: 100%;
      max-width: 500px;
      max-height: 85vh;
    }
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: space-between;
    flex-direction: column;
    border-radius: 0 0 20px 20px;
    border-top: none;
  }

  .messages {
    padding: 60px 0 0;
    flex: 1 1 auto;
    overflow-y: scroll;
    overflow-x: hidden;

    .messages-content {
      padding: 0 15px;
      width: 100%;
      height: 100%;

      .message {
        @include tablet {
          font-size: 22px;

          .message-definition {
            font-size: 18px !important;
          }
        }
        @include desktop {
          font-size: 20px;
        
          .message-definition {
            font-size: 16px !important;
          }
        }
        position: relative;
        max-width: 310px;
        word-break: break-word;
        clear: both;
        float: left;
        padding: 6px 10px 7px;
        border-radius: 0 10px 10px 10px;
        background: $color-background;
        margin: 8px 0;
        font-size: 17px;
        line-height: 1.4;
        margin-left: 40px;
        color: $color-text;
        transform: scale(0);
        transform-origin: 0 0;
        animation: bounce 500ms linear both;
        
        .message-definition {
          font-size: 13px;
          text-align: left;
        }

        &.message-personal {
          float: right;
          text-align: right;
          border-radius: 10px 10px 0 10px;
          background: $color-default;
          color: $color-text-white;
          margin: 2px 0;
        }

        &.loading {
          width: 50px;
          height: 30px;
          &::before {
            @include ball;
            border: none;
            animation-delay: .15s;
          }

          & span {
            display: block;
            font-size: 0;
            width: 30px;
            height: 20px;
            position: relative;

            &::before {
              @include ball;
              margin-left: -8px;
            }

            &::after {
              @include ball;
              margin-left: 8px;
              animation-delay: .3s;
            }
          }
        }

        .avatar {
          @include flex-center;
          position: absolute;
          top: 0;
          left: -40px;
          width: 35px;
          height: 35px;
          border-radius: 50%;
          border: 1px solid $color-default;
        }
      }
    }
  }

  .message-box {
    flex: 0 1 45px;
    width: 90%;
    margin: 15px auto;
    padding: 8px 0 8px 15px;
    position: relative;
    height: 14px;
    border: 1px solid #ccc;
    border-radius: 40px;
    height: 100%;
    display: flex;
    align-items: center;

    input {
      background: none;
      border: none;
      outline: none;
      font-size: 15px;
      height: 24px;
      width: 85%;
      color: $color-text;
    }

    span {
      position: absolute;
      top: 50%;
      right: -5px;
      transform: translateY(-50%);
      border-radius: 50%;
      button {
        @include flex-center;
        border: none;
        outline: none;
        width: 60px;
        height: 60px;
        background: transparent;
        cursor: pointer;
      }
    }
  }

  .restart {
    cursor: pointer;
    flex: 0 1 45px;
    width: 90%;
    margin: 15px auto;
    padding: 8px 0 8px 15px;
    height: 100%;
    text-align: center;
    border-radius: 15px;
    background: $color-background;
    color: $color-default;

    &:hover {
      filter: brightness(95%);
    }
    
    .restart-name {
      font-size: 17px;
      font-weight: bold;
    }
  }
</style>