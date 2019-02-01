---
title: 為何 X 不實做 Hook?
date: '2019-01-26'
spoiler: Just because we can, doesn’t mean we should.
---

當第一個 Alpha 版本的 [React Hooks](https://reactjs.org/hooks) 被釋出, 這裡有一個馬上隨之而來的問題: 「為何 *\<某些其他 API\>* 不實作 Hook? 」



提醒你，底下這一些是*有*實作 Hooks的:

* [`useState()`](https://reactjs.org/docs/hooks-reference.html#usestate) 讓你聲明一個 state 變數。
* [`useEffect()`](https://reactjs.org/docs/hooks-reference.html#useeffect) 讓你聲明一個 side effect。
* [`useContext()`](https://reactjs.org/docs/hooks-reference.html#usecontext) 讓你閱讀一些 context。

但這裡有些 APIs, 像是 `React.memo()` 和 `<Context.Provider>`， 它們並沒有實作 Hooks。
通常，被提出會實作 Hook 版本的必須是 「*noncompositional*」或是「*antimodular*」。這篇文章將會幫助你理解為什麼。

**注意: 對於那些對API討論感興趣的人來說，這篇文章非常深入。你不需要考慮使用 React 來提高效率！**

---

底下有兩個重要的屬性，我們希望 React API 要保留：

1. **Composition:** [Custom Hooks](https://reactjs.org/docs/hooks-custom.html) 很大程度上是我們對Hooks API感到興奮的原因。我們希望人們經常建立自己的Hooks，我們需要確保不同人編寫的Hooks [不會發生衝突 don't conflict](/why-do-hooks-rely-on-call-order/#flaw-4-the-diamond-problem)。 (難道我們都不會因為組件 「能被乾淨的compose」 以及「不會互相破壞」而寵壞了嗎？)

2. **Debugging:** 我們希望隨著應用程序的增長，[很容易找到錯誤 easy to find](/the-bug-o-notation/)。 React的最佳功能之一是，如果你看到錯誤呈現的東西，你可以回到上一層樹的結構，直到找到哪個組件的支柱或狀態導致錯誤。

這兩個限制前題放在一起可以告訴我們什麼可以 或 *不能* 實作成 Hooks。 我們來試試幾個例子吧。

---

## 一個真正的 Hook: `useState()`

### Composition

不同的地方建立的 Hooks 各自使用 `useState()` 是不會衝突的。

```js
function useMyCustomHook1() {
  const [value, setValue] = useState(0);
  // What happens here, stays here.
}

function useMyCustomHook2() {
  const [value, setValue] = useState(0);
  // What happens here, stays here.
}

function MyComponent() {
  useMyCustomHook1();
  useMyCustomHook2();
  // ...
}
```

加入一個新的無條件的 `useState()` 是安全的。 您不需要了解組件其他使用 Hooks 的變量宣稱。當然您也沒辦法透過更新其中一個狀態來更新其他狀態的變數內容。

**判定:** ✅ `useState()` 不會使得建立的 Hooks 是脆弱的。

### Debugging

Hooks 非常有用，因為你可以在 Hooks 之間*互相傳遞值*:

```js{4,12,14}
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  // ...
  return width;
}

function useTheme(isMobile) {
  // ...
}

function Comment() {
  const width = useWindowWidth();
  const isMobile = width < MOBILE_VIEWPORT;
  const theme = useTheme(isMobile);
  return (
    <section className={theme.comment}>
      {/* ... */}
    </section>
  );
}
```

但是當我們犯了錯誤怎麼辦？除錯的故事又會怎麼樣呢？

假設我們從`theme.comment`獲得的CSS Class 是錯誤的。 我們如何Debug出這個問題？ 我們可以在組件中設置斷點或埋幾個log。

也許我們會看到`theme`是錯誤的但是`width`和`isMobile`是正確的。 這會告訴我們問題是在`useTheme（）`裡面。 或者也許我們會看到`width`本身是錯誤的。 那會告訴我們要研究`useWindowWidth（）`。

**單獨查看中間值會告訴我們頂層的哪些Hook包含錯誤。** 我們不需要查看所有其他Hooks的實作方式。

然後我們可以“放大”有錯誤的那個，並重複這個步驟。

如果自定義Hook嵌套的深度增加，這變得更加重要。 想像一下，我們有3個級別的自定義Hook嵌套，每個級別使用3個不同的自定義Hooks。尋找 **3處**與潛在檢查 **3 + 3×3 + 3×3×3 = 39處** 之間的 [差異](/--bug-o-notation/)是巨大的。幸運的是，`useState（）`不能神奇地「影響」其他Hook或組件。 它返回的一個錯誤值會在它後面留下一條痕跡，就像任何變量一樣。🐛

**判定:** ✅ `useState()` 不會掩蓋代碼中的因果關係。 我們可以直接跟踪麵包屑找到bug。

---

## 不是一個 Hook: `useBailout()`

為了優化, 使用 Hooks 的組件可以避免不必要的重繪。

One way to do it is to put a [`React.memo()`](https://reactjs.org/blog/2018/10/23/react-v-16-6.html#reactmemo) wrapper around the whole component. It bails out of re-rendering if props are shallowly equal to what we had during the last render. This makes it similar to `PureComponent` in classes.

`React.memo()` takes a component and returns a component:

```js{4}
function Button(props) {
  // ...
}
export default React.memo(Button);
```

**But why isn’t it just a Hook?**

Whether you call it `useShouldComponentUpdate()`, `usePure()`, `useSkipRender()`, or `useBailout()`, the proposal tends to look something like this:

```js
function Button({ color }) {
  // ⚠️ Not a real API
  useBailout(prevColor => prevColor !== color, color);

  return (
    <button className={'button-' + color}>  
      OK
    </button>
  )
}
```

There are a few more variations (e.g. a simple `usePure()` marker) but in broad strokes they have the same flaws.

### Composition

Let’s say we try to put `useBailout()` in two custom Hooks:

```js{4,5,19,20}
function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  // ⚠️ Not a real API
  useBailout(prevIsOnline => prevIsOnline !== isOnline, isOnline);

  useEffect(() => {
    const handleStatusChange = status => setIsOnline(status.isOnline);
    ChatAPI.subscribe(friendID, handleStatusChange);
    return () => ChatAPI.unsubscribe(friendID, handleStatusChange);
  });

  return isOnline;
}

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  
  // ⚠️ Not a real API
  useBailout(prevWidth => prevWidth !== width, width);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });

  return width;
}
```

Now what happens if you use them both in the same component?


```js{2,3}
function ChatThread({ friendID, isTyping }) {
  const width = useWindowWidth();
  const isOnline = useFriendStatus(friendID);
  return (
    <ChatLayout width={width}>
      <FriendStatus isOnline={isOnline} />
      {isTyping && 'Typing...'}
    </ChatLayout>
  );
}
```

When does it re-render?

If every `useBailout()` call has the power to skip an update, then updates from `useWindowWidth()` would be blocked by `useFriendStatus()`, and vice versa. **These Hooks would break each other.**

However, if `useBailout()` was only respected when *all* calls to it inside a single component “agree” to block an update, our `ChatThread` would fail to update on changes to the `isTyping` prop.

Even worse, with these semantics **any newly added Hooks to `ChatThread` would break if they don’t *also* call `useBailout()`**. Otherwise, they can’t “vote against” the bailout inside `useWindowWidth()` and `useFriendStatus()`.

**Verdict:** 🔴 `useBailout()` breaks composition. Adding it to a Hook breaks state updates in other Hooks. We want the APIs to be [antifragile](/optimized-for-change/), and this behavior is pretty much the opposite.

### Debugging

How does a Hook like `useBailout()` affect debugging?

We’ll use the same example:

```js
function ChatThread({ friendID, isTyping }) {
  const width = useWindowWidth();
  const isOnline = useFriendStatus(friendID);
  return (
    <ChatLayout width={width}>
      <FriendStatus isOnline={isOnline} />
      {isTyping && 'Typing...'}
    </ChatLayout>
  );
}
```

Let’s say the `Typing...` label doesn’t appear when we expect, even though somewhere many layers above the prop is changing. How do we debug it?

**Normally, in React you can confidently answer this question by looking *up*.** If `ChatThread` doesn’t get a new `isTyping` value, we can open the component that renders `<ChatThread isTyping={myVar} />` and check `myVar`, and so on. At one of these levels, we’ll either find a buggy `shouldComponentUpdate()` bailout, or an incorrect `isTyping` value being passed down. One look at each component in the chain is usually enough to locate the source of the problem.

However, if this `useBailout()` Hook was real, you would never know the reason an update was skipped until you checked *every single custom Hook* (deeply) used by our `ChatThread` and components in its owner chain. Since every parent component can *also* use custom Hooks, this [scales](/the-bug-o-notation/) terribly.

It’s like if you were looking for a screwdriver in a chest of drawers, and each drawer contained a bunch of smaller chests of drawers, and you don’t know how deep the rabbit hole goes.

**Verdict:** 🔴 Not only `useBailout()` Hook breaks composition, but it also vastly increases the number of debugging steps and cognitive load for finding a buggy bailout — in some cases, exponentially.

---

We just looked at one real Hook, `useState()`, and a common suggestion that is intentionally *not* a Hook — `useBailout()`. We compared them through the prism of Composition and Debugging, and discussed why one of them works and the other one doesn’t.

While there is no “Hook version” of `memo()` or `shouldComponentUpdate()`, React *does* provide a Hook called [`useMemo()`](https://reactjs.org/docs/hooks-reference.html#usememo). It serves a similar purpose, but its semantics are different enough to not run into the pitfalls described above.

`useBailout()` is just one example of something that doesn’t work well as a Hook. But there are a few others — for example, `useProvider()`, `useCatch()`, or `useSuspense()`.

Can you see why?

*(Whispers: Composition... Debugging...)*
