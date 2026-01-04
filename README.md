# WebSocket_configure


# Service 

```jsx 
import { getAuthData } from "../../../config/Config";

// ........................Get Auth Token.......................... //



// ------------------ **WebSocket Connection for Chat List** ------------------ //
export const connectWebSocketForChatList = ({ onMessage, onSeen }) => {

    let token = null;
    try {
        const { accessToken } = getAuthData();
        token = accessToken;
    } catch (e) {
        console.error("Failed to get auth data:", e);
        return null; // stop if token not found
    }
    if (!token) {
        console.error("No access token found, cannot connect WebSocket");
        return null;
    }

    const wsProtocol = window.location.protocol === "https:" ? "wss" : "ws";
    const wsUrl = `${wsProtocol}://10.10.13.2:8000/ws/rooms/?token=${token}`;
    const socket = new WebSocket(wsUrl);
    socket.onopen = () => console.log("✅ WebSocket connected for Chat List");


    socket.onmessage = (event) => {
        try {
            const data = JSON.parse(event.data);
            console.log("📩 WS message:", data);

            switch (data.type) {
                case "chat_message":
                    onMessage?.(data.message ?? data)
                    break

                case "messages_seen_update":
                    onSeen?.(data.message_ids, data.seen_by)
                    break

                default:
                    console.warn("Unknown WS event type:", data.type)
                    onMessage?.(data)
            }
        } catch (err) {
            console.error("Failed to parse WebSocket message:", err)
        }
    };


    socket.onclose = () =>
        console.log("❌ WebSocket disconnected for Chat List");
    socket.onerror = (e) => console.error("⚠️ WebSocket error", e);

    return socket;
};


```

# Connect the socket on specific component
```jsx

  const [chatList, setChatList] = useState([])
    const socketRef = useRef(null);

    useEffect(() => {
         socketRef.current = connectWebSocketForChatList({
            onMessage: (message) => {
                console.log("New chat message received:", message)
                setChatList((prevList) => [message, ...prevList])
            },
            onSeen: (messageIds, seenBy) => {
                console.log("Messages seen update:", messageIds, seenBy)
                setChatList((prevList) =>
                    prevList.map((chat) =>
                        messageIds.includes(chat.id)
                            ? { ...chat, seen_by: [...(chat.seen_by || []), ...seenBy] }
                            : chat
                    )
                )
            },
        })

        return () => {
            if (socketRef.current) {
                socketRef.current.close()
            }
        }
    }, [])
```

# Update new mast on cach on tanstack query
```jsx
 // ..............**Connecting to WebSocket**..................\\
  useEffect(() => {
    if (!chatRoom) return;

    socketRef.current = connectWebSocketForChat({
      roomId: chatRoom,

      onMessage: (message) => {
        console.log("New msg arives Chatpanel:", message)
        if (message.type !== 'message') return;

        const newMessage = message.data;
        // Updated messages and cache with new message
        queryClient.setQueryData(
          ['messages', chatRoom],
          (oldMessages = []) => {
            // Safety: ensure array
            if (!Array.isArray(oldMessages)) return oldMessages;

            // Prevent duplicate messages
            const alreadyExists = oldMessages.some(
              msg => msg.id === newMessage.id
            );

            if (alreadyExists) return oldMessages;

            // Add new message to TOP (newest first)
            return [newMessage, ...oldMessages];
          }
        );
      },
    });

    return () => {
      socketRef.current?.close();
    };
  }, [chatRoom, queryClient]);
```
# Messages fetching...........
```jsx 
"use client";

import { useInfiniteQuery, useQueryClient } from "@tanstack/react-query";
import { useState, useEffect, useMemo } from "react";
import { FiSend, FiInfo } from "react-icons/fi";
import axiosApi from "../../../service/axiosInstance";
import { connectWebSocketForChat } from "./ChatService";
import { getAuthData } from "../../../config/Config";
import MessageList from "./MessageList";

const ChatPanel = ({ chatRoom, currentUser }) => {
  const queryClient = useQueryClient();
  const [inputMessage, setInputMessage] = useState("");
  const [showActions, setShowActions] = useState(false);

  // Auth
  const { userInfo } = getAuthData();
  const userId = userInfo?.user_id;



  // Messages (HTTP with infinite scroll)
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ["messages", chatRoom],
    enabled: !!chatRoom,
    queryFn: async ({ pageParam = null }) => {
      const res = await axiosApi.get(
        `/api/v1/rooms/${chatRoom}/messages/`,
        { params: { cursor: pageParam } }
      );

      return {
        results: res.data.results ?? [],
        nextCursor: res.data.next_cursor ?? null,
        chatInfo: res.data.room ?? null,
      };
    },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  console.log("Result--------------:", data)
  console.log("Result*************:", data?.pages[0].chatInfo)


  // Flatten and reverse to show oldest -> newest
  const messages = useMemo(() => {
    const list = data?.pages.flatMap((p) => p.results) ?? [];
    return [...list].reverse();
  }, [data]);

  console.log("Total Messages :", messages)

  // WebSocket for real-time messages
  useEffect(() => {
    if (!chatRoom) return;

    const socket = connectWebSocketForChat({
      roomId: chatRoom,

      onMessage: (payload) => {
        if (payload.type !== "message") return;
        const newMessage = payload.data;
        console.log("New messages:", newMessage)

        queryClient.setQueryData(["messages", chatRoom], (old) => {
          if (!old) return old;

          const exists = old.pages.some((p) =>
            p.results.some((m) => m.id === newMessage.id)
          );

          if (exists) return old;

          console.log("Adding new message to FIRST page (newest messages)")

          // Always add new messages to the FIRST page (page 0), not last page
          // First page = newest messages, last page = oldest messages
          return {
            ...old,
            pages: old.pages.map((p, i) =>
              i === 0
                ? { ...p, results: [newMessage, ...p.results] }
                : p
            ),
          };
        });
      },
    });

    return () => socket.close();
  }, [chatRoom, queryClient]);

  // Send message with optimistic update
  const handleSendMessage = async () => {
    if (!inputMessage.trim()) return;

    const optimisticMsg = {
      id: `temp-${Date.now()}`,
      text: inputMessage,
      created_at: new Date().toISOString(),
      sender: { id: userId },
      avatar: safeUser.avatar,
    };

    setInputMessage("");
    // .....................** Send Messages **..................... //
    try {
      const formData = new FormData();
      // Just appent paylod fields to formData
      formData.append("content", inputMessage);

      await axiosApi.post(`/api/v1/rooms/${chatRoom}/send/`, formData, {
        headers: {
          "Content-Type": "multipart/form-data",
        },
      });
      console.log("Message send")
    } catch (err) {
      console.error("Send failed", err);
    }
  };
  // ...........................**Chat info**........................... //
  // data?.pages[0].chatInfo)
  const safeUser = {
    name: data?.pages[0]?.chatInfo?.name || "Unknocwn User",
    role: data?.pages[0]?.chatInfo?.display_role || "unknown",
    avatar: `http://10.10.13.2:8000${data?.pages[0]?.chatInfo?.image}` ||
      "https://api.dicebear.com/7.x/avataaars/svg?seed=Chat",
  };

  return (
    <div className="flex flex-col h-full border border-gray-300 rounded-lg bg-white">
      {!chatRoom ? (
        <div className="flex-1 flex items-center justify-center text-gray-500">
          Select a chat
        </div>
      ) : (
        <>
          {/* Header */}
          <div className="p-4 border-b border-gray-300 flex justify-between items-center">
            <div className="flex gap-3 items-center">
              <img
                src={safeUser.avatar}
                className="w-10 h-10 rounded-full"
                alt=""
              />
              <div>
                <div className="font-semibold">{safeUser.name}</div>
                <div className="text-xs text-pink-600">{safeUser.role}</div>
              </div>
            </div>

            <FiInfo
              className="cursor-pointer"
              onClick={() => setShowActions(!showActions)}
            />
          </div>

          {/* Message List */}
          <MessageList
            messages={messages}
            userId={userId}
            onLoadMore={fetchNextPage}
            hasNextPage={hasNextPage}
            isFetchingNextPage={isFetchingNextPage}
          />

          {/* Input Area */}
          <div className="p-4 border-t border-gray-300">
            <div className="flex gap-3">
              <input
                value={inputMessage}
                onChange={(e) => setInputMessage(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && handleSendMessage()}
                placeholder="Type a message..."
                className="flex-1 outline-none"
              />
              <button
                onClick={handleSendMessage}
                disabled={!inputMessage.trim()}
              >
                <FiSend size={24} />
              </button>
            </div>
          </div>
        </>
      )}
    </div>
  );
};

export default ChatPanel;
```
# MessageList
```jsx
import { useEffect, useRef, useState } from "react";

const MessageList = ({ 
  messages, 
  userId, 
  onLoadMore, 
  hasNextPage, 
  isFetchingNextPage 
}) => {
  const messagesEndRef = useRef(null);
  const containerRef = useRef(null);
  const isLoadingRef = useRef(false);
  const prevScrollHeightRef = useRef(0);
  const [prevMessageCount, setPrevMessageCount] = useState(0);
  const wasAtBottomBeforeFetchRef = useRef(true);

  // Auto-scroll to bottom ONLY if user was already viewing the bottom (new real-time messages)
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    // Never scroll during pagination
    if (isFetchingNextPage) {
      return;
    }

    // Check if new messages arrived
    const messageCountChanged = messages.length !== prevMessageCount;
    
    if (messageCountChanged && wasAtBottomBeforeFetchRef.current) {
      // Only scroll to bottom if new messages were added AND user was viewing the bottom
      const messageDifference = messages.length - prevMessageCount;
      
      if (messageDifference > 0) {
        // New messages arrived and user was at bottom - scroll to bottom
        setTimeout(() => {
          container.scrollTop = container.scrollHeight - container.clientHeight;
        }, 0);
      }
      
      setPrevMessageCount(messages.length);
    } else if (messageCountChanged) {
      // Message count changed but user wasn't at bottom (pagination) - just update count
      setPrevMessageCount(messages.length);
    }
  }, [messages.length, isFetchingNextPage, prevMessageCount]);

  // Restore scroll position after older messages load
  useEffect(() => {
    const container = containerRef.current;
    if (!container || isFetchingNextPage) return;

    // After fetch completes, restore scroll position
    if (prevScrollHeightRef.current > 0) {
      const newScrollHeight = container.scrollHeight;
      const heightDifference = newScrollHeight - prevScrollHeightRef.current;
      
      // Scroll down by the height of newly added messages to stay in same visual position
      container.scrollTop = heightDifference;
      console.log("📍 Scroll position restored - added", heightDifference, "px");
      
      prevScrollHeightRef.current = 0;
      wasAtBottomBeforeFetchRef.current = false; // User scrolled to top for pagination, not at bottom
    }
  }, [isFetchingNextPage]);

  // Infinite scroll - load older messages when scroll to top
  const handleScroll = (e) => {
    const { scrollTop, scrollHeight, clientHeight } = e.target;

    // Track if user is at the bottom (for auto-scroll later)
    const isAtBottom = scrollHeight - scrollTop - clientHeight < 50;
    wasAtBottomBeforeFetchRef.current = isAtBottom;

    // When user scrolls to top
    if (scrollTop === 0 && hasNextPage && !isFetchingNextPage && !isLoadingRef.current) {
      isLoadingRef.current = true;
      console.log("📍 Top reached - loading older messages");
      
      // Store current scroll position before fetch
      prevScrollHeightRef.current = e.target.scrollHeight;
      
      onLoadMore();

      // Reset flag after a delay
      setTimeout(() => {
        isLoadingRef.current = false;
      }, 500);
    }
  };

  const MessageBubble = ({ msg }) => {
    const isMe = Number(msg?.sender?.id) === Number(userId);
    const text = msg.text || msg.message || msg.content || "";

    return (
      <div className={`flex mb-4 ${isMe ? "justify-end" : "justify-start"}`}>
        <div
          className={`px-4 py-2 rounded-lg max-w-md ${
            isMe ? "bg-teal-100 text-gray-900" : "bg-blue-100 text-gray-900"
          }`}
        >
          <p className="text-sm">{text}</p>
          <div className="text-xs text-gray-500 mt-1">
            {new Date(msg.created_at).toLocaleTimeString([], {
              hour: "2-digit",
              minute: "2-digit",
            })}
          </div>
        </div>
      </div>
    );
  };

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      className="flex-1 overflow-y-auto p-4 space-y-2"
    >
      {isFetchingNextPage && (
        <div className="text-center text-sm text-gray-500 py-2">
          Loading older messages...
        </div>
      )}

      {messages.map((msg) => (
        <MessageBubble key={msg.id} msg={msg} />
      ))}

      <div ref={messagesEndRef} />
    </div>
  );
};

export default MessageList;


