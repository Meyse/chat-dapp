<script lang="ts">
// Component: src/lib/components/ChatInterface.svelte
// Description: Main container for the VerusID chat interface.
// Manages the overall layout, conversations, messages, polling, and sending.
// Changes:
// - Removed mock data initialization for conversations and messages.
// - Initialized conversations and messages state as empty.
// - Added periodic polling for new messages using setInterval.
// - Implemented message merging logic, preventing duplicates.
// - Added unread indicator logic for conversations.
// - Refined sorting logic to prioritize confirmations for received messages.
// - Imported global Conversation type from types.ts.
// - Stored recipient_private_address in conversations array.
// - Implemented handleSendMessage to perform optimistic UI update and call backend.
// - Corrected message sorting to be descending by timestamp.
// - Added chat persistence logic (prompt, load/save settings, convos, messages).
// - Added PersistencePromptModal.
// - Added SettingsView integration.

  import { createEventDispatcher, onMount, onDestroy } from 'svelte';
  import { invoke } from '@tauri-apps/api/core';
  import TopBar from './chat/TopBar.svelte';
  import ConversationsList from './chat/ConversationsList.svelte';
  import ConversationView from './chat/ConversationView.svelte';
  import NewChatModal from './chat/NewChatModal.svelte';
  import PersistencePromptModal from './chat/PersistencePromptModal.svelte'; // Import persistence modal
  import SettingsView from './settings/SettingsView.svelte'; // Import settings view
  import type { FormattedIdentity, PrivateBalance, ChatMessage, Conversation } from '$lib/types';

  // --- Props ---
  export let loggedInIdentity: FormattedIdentity | null = null; // Received from parent (+page.svelte)
  export let blockHeight: number | null = null; // Received from parent (+page.svelte)
  export let privateBalance: PrivateBalance = null; // Use the new type alias

  // --- Constants ---
  const POLLING_INTERVAL_MS = 30000; // 30 seconds

  // --- Event dispatcher ---
  const dispatch = createEventDispatcher<{
    logout: void;
  }>();

  // --- State ---
  let selectedConversationId: string | null = null; // Use recipient's formatted_name as ID
  let showNewChatModal = false;
  let showSettingsView = false; // NEW: Track if settings view is active

  // Pending Transaction State
  let isTransactionPending = false;
  let pendingSinceBlock: number | null = null;

  // Persistence State
  let showPersistencePrompt = false;
  let persistenceSetting: boolean | null = null; // null = setting not yet determined/loaded

  // --- Data State (Initialized Empty) ---
  let conversations: Conversation[] = []; // Use the imported type
  let messages: { [conversationId: string]: ChatMessage[] } = {};
  let pollingIntervalId: ReturnType<typeof setInterval> | null = null;
  let isPolling = false;
  let selectedMessages: ChatMessage[] = []; // Add declaration for computed messages

  // --- Computed State ---
  $: existingConversationIds = conversations.map(c => c.id.toLowerCase());
  $: loggedInUserPrivateAddress = loggedInIdentity?.private_address;
  $: loggedInUserIdentityName = loggedInIdentity?.formatted_name;
  $: loggedInUserIAddress = loggedInIdentity?.i_address; // Needed for persistence keys

  // --- Lifecycle ---
  onMount(() => {
      console.log("ChatInterface: onMount - Logged in ID:", loggedInIdentity);
      // Ensure initial sort if data is loaded
      checkAndApplyPersistence(); 
  });

  onDestroy(() => {
      stopPolling();
  });

  // --- Persistence Logic ---
  async function checkAndApplyPersistence() {
      if (!loggedInUserIAddress) {
          console.error("ChatInterface: Cannot check persistence, logged in user iAddress not available.");
          startPolling(); // Start polling even if persistence fails (ephemeral mode)
          return;
      }

      console.log(`ChatInterface: Checking persistence setting for ${loggedInUserIAddress}...`);
      try {
          const setting = await invoke<boolean | null>('load_persistence_setting', { 
              identityIAddress: loggedInUserIAddress 
          });
          console.log(`ChatInterface: Persistence setting loaded: ${setting}`);

          if (setting === null) {
              // No setting found, prompt the user
              console.log("ChatInterface: No persistence setting found. Prompting user.");
              persistenceSetting = null; // Explicitly null
              showPersistencePrompt = true;
              // Do not load data or start polling yet, wait for user choice
          } else {
              persistenceSetting = setting;
              if (setting === true) {
                  console.log("ChatInterface: Persistence enabled. Loading saved data...");
                  await loadSavedData();
              } else {
                  console.log("ChatInterface: Persistence disabled. Initializing empty state.");
                  conversations = [];
                  messages = {};
              }
              // Start polling after setting is determined and initial data (if any) is loaded
              startPolling(); 
          }
      } catch (error) {
          console.error("ChatInterface: Error loading persistence setting:", error);
          persistenceSetting = false; // Default to false on error
          conversations = []; // Ensure clean state on error
          messages = {};
      }
  }

  async function loadSavedData() {
       if (!loggedInUserIAddress) return;

       try {
           console.log("ChatInterface: Loading conversations...");
           const savedConversations = await invoke<Conversation[]>('load_conversations', {
               identityIAddress: loggedInUserIAddress
           });
           conversations = savedConversations || []; // Ensure it's an array
           console.log(`ChatInterface: Loaded ${conversations.length} conversations.`);

           // Load messages for all loaded conversations
           const loadedMessages: { [conversationId: string]: ChatMessage[] } = {};
           if (conversations.length > 0) {
               console.log("ChatInterface: Loading messages for loaded conversations...");
               for (const convo of conversations) {
                   try {
                       const savedMessages = await invoke<ChatMessage[]>('load_messages_for_conversation', {
                           identityIAddress: loggedInUserIAddress,
                           conversationId: convo.id
                       });
                       loadedMessages[convo.id] = sortMessages(savedMessages || [], blockHeight); 
                       console.log(`ChatInterface: Loaded ${loadedMessages[convo.id].length} messages for ${convo.id}`);
                   } catch (msgError) {
                       console.error(`ChatInterface: Error loading messages for conversation ${convo.id}:`, msgError);
                       loadedMessages[convo.id] = []; // Initialize empty on error for this convo
                   }
               }
           }
           messages = loadedMessages;
           console.log("ChatInterface: Finished loading all saved message data.");

           // Trigger reactivity for lists
           conversations = [...conversations];
           messages = {...messages};

       } catch (error) {
           console.error("ChatInterface: Error loading saved conversations:", error);
           conversations = []; // Reset on error
           messages = {};
       }
  }

  async function handleSavePreference(save: boolean) {
      if (!loggedInUserIAddress) {
          console.error("ChatInterface: Cannot save persistence preference, logged in user iAddress not available.");
          showPersistencePrompt = false; // Hide prompt anyway
          startPolling(); // Start ephemeral polling
          return;
      }
      
      console.log(`ChatInterface: User chose to ${save ? 'save' : 'not save'} persistence.`);
      try {
          await invoke('save_persistence_setting', {
              identityIAddress: loggedInUserIAddress,
              savePreference: save
          });
          persistenceSetting = save;
          showPersistencePrompt = false;

          if (save) {
              // If they chose YES, ensure state is ready (should be empty)
              conversations = [];
              messages = {};
              // We don't load data here because they just opted *in* - there's nothing *to* load yet.
          } else {
              // If they chose NO, ensure state is empty
              conversations = [];
              messages = {};
          }
           // Start polling now that preference is set
          startPolling();

      } catch (error) {
          console.error("ChatInterface: Error saving persistence setting:", error);
          // TODO: Show error to user?
          showPersistencePrompt = false; // Hide prompt even on error
          persistenceSetting = false; // Assume false on save error
          startPolling(); // Start ephemeral polling on error
      }
  }

  // --- Polling Logic ---
  function startPolling() {
      // Only start polling if persistence setting is known and we have an address
      if (persistenceSetting === null || !loggedInUserPrivateAddress) {
          console.log(`ChatInterface: Delaying polling start. Persistence Setting Known: ${persistenceSetting !== null}, Address available: ${!!loggedInUserPrivateAddress}`);
          return;
      }
      stopPolling(); // Ensure no duplicate intervals
      console.log(`ChatInterface: Starting message polling every ${POLLING_INTERVAL_MS / 1000}s`);
      fetchNewMessages(); // Initial fetch
      pollingIntervalId = setInterval(fetchNewMessages, POLLING_INTERVAL_MS);
  }

  function stopPolling() {
      if (pollingIntervalId) {
          console.log("ChatInterface: Stopping message polling.");
          clearInterval(pollingIntervalId);
          pollingIntervalId = null;
      }
      isPolling = false;
  }

  async function fetchNewMessages() {
      if (!loggedInUserPrivateAddress || isPolling) {
          if (isPolling) console.log("ChatInterface: Skipping poll, already fetching.");
          return; 
      }
      
      isPolling = true;
      console.log("ChatInterface: Polling for new messages...");
      
      try {
          const newMessages = await invoke<ChatMessage[]>('get_new_received_messages', { 
              ownPrivateAddress: loggedInUserPrivateAddress 
          });
          
          console.log(`ChatInterface: Received ${newMessages.length} potential new messages from poll.`);
          if (newMessages.length > 0) {
              processNewMessages(newMessages);
          }

      } catch (error) {
          console.error("ChatInterface: Error polling for new messages:", error);
          // Consider stopping polling on certain errors? Or just log?
      } finally {
          isPolling = false;
      }
  }

  function processNewMessages(newMessages: ChatMessage[]) {
      let messagesUpdated = false;
      let conversationsUpdated = false;
      const currentMessages = { ...messages }; // Clone to modify

      for (const newMessage of newMessages) {
          const senderId = newMessage.sender; // Sender ID from memo
          
          // Check if we have an existing conversation with this sender
          const convoIndex = conversations.findIndex(c => c.id === senderId);
          if (convoIndex !== -1) {
              // Conversation exists, check if message is new
              const currentList = currentMessages[senderId] || [];
              const existingMsgIndex = currentList.findIndex(m => m.id === newMessage.id);
              
              if (existingMsgIndex === -1) {
                  // New message for this known conversation
                  console.log(`ChatInterface: Adding new message ${newMessage.id} from ${senderId}`);
                  currentList.push(newMessage);
                  // Sort only the affected list
                  currentList.sort(compareMessages);
                  currentMessages[senderId] = currentList; // Update the list in our temporary object
                  messagesUpdated = true;

                  // ----> SAVE MESSAGE IF PERSISTENCE ENABLED <----
                  if (persistenceSetting === true && loggedInUserIAddress) {
                      saveMessages(loggedInUserIAddress, senderId, currentList);
                  }
                  // <-----------------------------------------------

                  // Mark conversation as unread if it's not the currently selected one
                  if (senderId !== selectedConversationId) {
                      if (!conversations[convoIndex].unread) {
                           console.log(`ChatInterface: Marking conversation ${senderId} as unread.`);
                           conversations[convoIndex].unread = true;
                           conversationsUpdated = true;
                      }
                  }
              } else {
                 // Message already exists, potentially update confirmations if needed (optional)
                 if (currentList[existingMsgIndex].confirmations !== newMessage.confirmations) {
                     console.log(`ChatInterface: Updating confirmations for message ${newMessage.id} from ${senderId}`);
                     currentList[existingMsgIndex].confirmations = newMessage.confirmations;
                     // No need to resort on confirmation update alone unless desired
                     messagesUpdated = true; // Mark as updated if confirmations changed
                 }
              }
          } else {
              // Conversation does not exist, ignore message as per requirements
              // console.log(`ChatInterface: Ignoring message ${newMessage.id} from unknown sender ${senderId}`);
          }
      }

      if (messagesUpdated) {
          // We don't need to resort everything here anymore, sorting happens per-conversation list
          messages = currentMessages; // Assign the updated object back to trigger reactivity
          console.log("ChatInterface: Messages state updated.");
      }
      if (conversationsUpdated) {
          conversations = [...conversations]; // Trigger reactivity for conversations list
          console.log("ChatInterface: Conversations state updated (unread).", conversations);
      }
  }

  // Comparison function for sorting messages
  function compareMessages(a: ChatMessage, b: ChatMessage): number {
    // Primary sort: Timestamp descending (newest first in array)
    const timestampDiff = b.timestamp - a.timestamp;
    if (timestampDiff !== 0) {
        return timestampDiff;
    }

    // Secondary sort (tie-breaker, e.g., for received messages with timestamp 0):
    // Confirmations ascending (more confirmations = older, comes later in descending sort)
    // This maintains relative order for messages potentially received in the same block
    if (a.direction === 'received' && b.direction === 'received') {
        return a.confirmations - b.confirmations; 
    }

    // If timestamps are identical and not both received, maintain original order (or arbitrary)
    return 0;
  }

  // --- Event Handlers ---
  function handleSelectConversation(event: CustomEvent<{ conversationId: string }>) {
    // Close settings if open when selecting a conversation
    if (showSettingsView) {
        showSettingsView = false;
    }
    selectedConversationId = event.detail.conversationId;
    console.log("ChatInterface: Selected conversation", selectedConversationId);
    const convoIndex = conversations.findIndex(c => c.id === selectedConversationId);
    if (convoIndex !== -1) {
      // Mark as read when selected
      if (conversations[convoIndex].unread) {
          conversations[convoIndex].unread = false;
          conversations = [...conversations]; // Trigger reactivity
      }
    }
    // TODO: Potentially load messages here if lazy loading is implemented later
  }

  async function handleSendMessage(event: CustomEvent<{ message: string; amount?: number }>) {
    const { message: messageText, amount } = event.detail;
    console.log("ChatInterface: Send message request", event.detail);

    // Ensure we have all necessary info
    if (!selectedConversationId || !loggedInIdentity || !loggedInUserPrivateAddress || !loggedInUserIdentityName) {
        console.error("ChatInterface: Cannot send message. Missing required data (selected convo ID, logged-in identity/address).");
        return;
    }

    // Find the recipient's private address from the conversation data
    const recipientConversation = conversations.find(c => c.id === selectedConversationId);
    if (!recipientConversation || !recipientConversation.recipient_private_address) {
        console.error(`ChatInterface: Cannot send message. Recipient private address not found for conversation ${selectedConversationId}.`);
        return;
    }
    const recipientPrivateAddress = recipientConversation.recipient_private_address;

    // 1. Optimistic Update
    const newMessage: ChatMessage = {
        id: `msg-${Date.now()}-${Math.random()}`, // Temporary unique ID
        sender: 'self',
        text: messageText,
        timestamp: Date.now(), // Keep for potential future use/tie-breaking?
        sentAtBlockHeight: blockHeight || 0, // Store block height when sent (use 0 if unavailable)
        status: 'sent', // Initial optimistic status
        amount: amount || 0,
        confirmations: 0, // Sent messages have 0 confirmations locally
        direction: 'sent'
    };

    // Get the current list or start with an empty array
    const currentList = messages[selectedConversationId] || [];
    // Create a *new* array with the new message added
    const newList = [...currentList, newMessage];
    
    // Sort the new list using the current block height
    messages[selectedConversationId] = sortMessages(newList, blockHeight);
    
    // Trigger top-level reactivity by creating a new messages object reference
    messages = { ...messages }; 

    console.log(`ChatInterface: Optimistically updated messages state for ${selectedConversationId}. New message ID: ${newMessage.id}`);

    // ----> SAVE MESSAGES IF PERSISTENCE ENABLED <----
    if (persistenceSetting === true && loggedInUserIAddress) {
        // Save the entire updated list for this conversation
        saveMessages(loggedInUserIAddress, selectedConversationId, messages[selectedConversationId]);
    }
    // <-----------------------------------------------

    // 2. Call Backend
    try {
        console.log("ChatInterface: Invoking send_private_message backend command...");
        const opid = await invoke<string>('send_private_message', {
            senderZAddress: loggedInUserPrivateAddress,
            recipientZAddress: recipientPrivateAddress,
            memoText: messageText,
            senderIdentity: loggedInUserIdentityName, // e.g., user@
            amount: amount || 0, 
        });
        console.log(`ChatInterface: Message send initiated successfully via backend. OPID: ${opid}`);

        // ---> Set Pending State on Success <----
        console.log(`ChatInterface: Setting pending state. Current block height: ${blockHeight}`);
        isTransactionPending = true;
        pendingSinceBlock = blockHeight; // Capture the block height *at the time of sending*
        // <------------------------------------

    } catch (error) {
        console.error("ChatInterface: Error sending message via backend:", error);
        // Revert optimistic update or mark message as failed
        const optimisticMessageIndex = messages[selectedConversationId]?.findIndex(m => m.id === newMessage.id);
        if (optimisticMessageIndex !== undefined && optimisticMessageIndex > -1) {
            messages[selectedConversationId][optimisticMessageIndex].status = 'failed';
            messages = { ...messages }; // Trigger reactivity to show potential failure
            console.log(`ChatInterface: Marked optimistic message ${newMessage.id} as failed.`);
        }
        // Ensure pending state is NOT set or is cleared on error
        isTransactionPending = false; 
        pendingSinceBlock = null;
    }
  }

  function handleOpenNewChatModal() {
    // Close settings if open when opening new chat modal
    if (showSettingsView) {
        showSettingsView = false;
    }
    console.log("ChatInterface: Open new chat modal requested");
    showNewChatModal = true;
  }

  function handleCloseNewChatModal() {
      showNewChatModal = false;
  }

  async function handleStartChat(event: CustomEvent<{ identity: FormattedIdentity; history?: ChatMessage[] }>) {
    const { identity, history } = event.detail;
    const newConversationId = identity.formatted_name; // Use the unique formatted name as ID
    console.log(`ChatInterface: Start chat event received for ${newConversationId}. History included: ${history ? history.length : 0}`, identity);

    // Check if recipient has a private address (essential for sending back)
    if (!identity.private_address) {
        console.error(`ChatInterface: Cannot start chat with ${newConversationId}, recipient has no private address.`);
        showNewChatModal = false; // Close modal on error
        // TODO: Show an error to the user?
        return; // Prevent adding conversation
    }

    let conversationsUpdated = false;
    let unreadStatusChanged = false;
    let messagesUpdated = false; // Track if messages were actually merged

    // 1. Add conversation (if it doesn't exist)
    const existingConvoIndex = conversations.findIndex(c => c.id === newConversationId);
    if (existingConvoIndex === -1) {
        const newConversation: Conversation = {
            id: newConversationId,
            name: identity.formatted_name,
            recipient_private_address: identity.private_address,
            unread: false // Start as read since we are selecting it
        };
        conversations = [...conversations, newConversation];
        conversationsUpdated = true;
        console.log(`ChatInterface: Added new conversation:`, newConversation);
    }

    // 2. Merge fetched history (if provided)
    let currentMessages = messages[newConversationId] || [];
    if (history && history.length > 0) { 
        const existingIds = new Set(currentMessages.map(m => m.id));
        const newHistoryMessages = history.filter(hm => !existingIds.has(hm.id));
        if (newHistoryMessages.length > 0) {
            currentMessages = [...newHistoryMessages, ...currentMessages]; // Add new history to the start
            messagesUpdated = true;
            console.log(`ChatInterface: Merged ${newHistoryMessages.length} history messages for ${newConversationId}.`);
        }
    }
    // Ensure messages are sorted after potential merge
    messages[newConversationId] = sortMessages(currentMessages, blockHeight); // Sort merged history
    messages = { ...messages }; // Trigger reactivity for messages

    // 3. Select the new conversation
    selectedConversationId = newConversationId;
    
    // 4. Ensure selected conversation is marked as read
    const finalConvoIndex = conversations.findIndex(c => c.id === newConversationId);
    if (finalConvoIndex !== -1 && conversations[finalConvoIndex].unread) {
        conversations[finalConvoIndex].unread = false;
        unreadStatusChanged = true; // Flag that we changed the status
        conversations = [...conversations]; // Trigger reactivity if changed
    }

    // ----> SAVE DATA IF PERSISTENCE ENABLED <----
    if (persistenceSetting === true && loggedInUserIAddress) {
        // Save the conversations list if we added a new one OR changed unread status
        if (conversationsUpdated || unreadStatusChanged) {
            saveConversationsList(); 
        }
        // Save the newly merged messages list if messages were updated
        if (messagesUpdated) {
             saveMessages(loggedInUserIAddress, newConversationId, messages[newConversationId]);
        }
    }
    // <-----------------------------------------------

    // 5. Close modal
    showNewChatModal = false;
  }

  // --- Helper Save Functions (with error handling) ---
  async function saveConversationsList() {
      if (!persistenceSetting || !loggedInUserIAddress) return;
      console.log("ChatInterface: Saving conversations list...");
      try {
          await invoke('save_conversations', {
              identityIAddress: loggedInUserIAddress,
              conversations: conversations // Pass the current full list
          });
      } catch (error) {
          console.error("ChatInterface: Error saving conversations list:", error);
          // TODO: User feedback?
      }
  }

  async function saveMessages(identityAddr: string, convoId: string, messagesToSave: ChatMessage[]) {
      if (!persistenceSetting) return;
      console.log(`ChatInterface: Saving ${messagesToSave.length} messages for ${convoId}...`);
      try {
          await invoke('save_messages_for_conversation', {
              identityIAddress: identityAddr,
              conversationId: convoId,
              messages: messagesToSave
          });
      } catch (error) {
          console.error(`ChatInterface: Error saving messages for ${convoId}:`, error);
          // TODO: User feedback?
      }
  }

  // --- Other Event Handlers ---
  function handleLogout() {
      console.log("ChatInterface: Logout requested");
      stopPolling(); // Stop polling on logout
      dispatch('logout'); 
  }
    
  function handleSettings() {
      console.log("ChatInterface: Settings requested");
      showSettingsView = !showSettingsView;
      // Deselect conversation when opening settings
      if (showSettingsView) {
          selectedConversationId = null;
      }
  }

  // --- Settings View Event Handlers ---

  async function handlePersistenceToggle(event: CustomEvent<{ enabled: boolean }>) {
      const enabled = event.detail.enabled;
      if (!loggedInUserIAddress) {
          console.error("ChatInterface: Cannot toggle persistence, logged in user iAddress not available.");
          return;
      }
      console.log(`ChatInterface: Toggling persistence setting to ${enabled} via Settings.`);
      try {
          await invoke('save_persistence_setting', {
              identityIAddress: loggedInUserIAddress,
              savePreference: enabled
          });
          persistenceSetting = enabled; // Update local state to match
          // If disabling, we might want to ask if they also want to delete data, but PRD implies separate actions
          if (!enabled) {
              console.warn("ChatInterface: Persistence disabled. Local data remains until explicitly deleted.")
              // Consider adding a toast notification here?
          }
      } catch (error) {
          console.error("ChatInterface: Error saving persistence setting from SettingsView:", error);
          // TODO: Show error to user (e.g., toast)
          // Revert visual toggle on error? Requires more complex state management in SettingsView
      }
  }

  async function handleDeleteHistory() {
      if (!loggedInUserIAddress) {
          console.error("ChatInterface: Cannot delete history, logged in user iAddress not available.");
          return;
      }
      console.warn(`ChatInterface: Deleting chat history for ${loggedInUserIAddress} via Settings.`);
      try {
          await invoke('delete_chat_data', { identityIAddress: loggedInUserIAddress });
          // Clear local state immediately
          conversations = [];
          messages = {};
          // Deselect conversation if one was somehow selected while settings were open
          selectedConversationId = null; 
          // Update conversations/messages reactively
          conversations = [...conversations];
          messages = {...messages};
          console.log("ChatInterface: Local chat state cleared after deletion.");
          // Optionally, could update persistenceSetting to null to force re-prompt, 
          // but PRD implies delete is separate from the preference setting itself.
          // persistenceSetting = null; 
      } catch (error) {
          console.error("ChatInterface: Error deleting chat data:", error);
          // TODO: Show error to user (e.g., toast)
      }
  }

  function handleCloseSettings() {
      showSettingsView = false;
  }

  // Reactive computation for selected conversation's messages
  $: {
    const rawMessages = selectedConversationId ? (messages[selectedConversationId] || []) : [];
    // Always ensure messages are sorted just before display
    selectedMessages = sortMessages(rawMessages, blockHeight);
  } 
  $: selectedContactName = showSettingsView ? null : selectedConversationId ? conversations.find(c => c.id === selectedConversationId)?.name : null; // Don't show contact name if settings are open

  // --- Reactive Logic for Clearing Pending State ---
  $: if (isTransactionPending && blockHeight !== null && pendingSinceBlock !== null && blockHeight > pendingSinceBlock) {
      console.log(`ChatInterface: Detected new block (${blockHeight} > ${pendingSinceBlock}). Clearing pending state.`);
      isTransactionPending = false;
      pendingSinceBlock = null;
  }

  // --- Sorting Logic (Using Effective Block Height) ---
  function sortMessages(messageList: ChatMessage[], currentBh: number | null): ChatMessage[] {
      if (currentBh === null) return messageList; // Cannot sort without current height

      return messageList.sort((a, b) => {
          const blockA = getEffectiveBlockHeight(a, currentBh);
          const blockB = getEffectiveBlockHeight(b, currentBh);

          // Handle nulls (messages where block couldn't be determined - place them older/top)
          if (blockA === null && blockB === null) return 0;
          if (blockA === null) return 1; 
          if (blockB === null) return -1;

          // Primary sort: Effective Block Height DESCENDING (newest block first in array)
          if (blockB !== blockA) {
              return blockB - blockA; 
          }

          // Secondary sort (tie-breaker for same block): 
          // Prefer Sent messages slightly earlier (lower in array, visually higher) than received in same block
          if (a.direction !== b.direction) {
               return a.direction === 'sent' ? -1 : 1;
          }

          // Tertiary sort (if both sent or both received in same block):
          if (a.direction === 'sent') {
              // Timestamp descending for sent (newest timestamp first)
              return (b.timestamp || 0) - (a.timestamp || 0); 
          } else {
              // Confirmations ascending for received (lowest confirmations first - means newer)
              return (a.confirmations ?? 0) - (b.confirmations ?? 0); 
          }
      });
  }

  function getEffectiveBlockHeight(message: ChatMessage, currentBh: number): number | null {
      if (message.direction === 'sent') {
          if (message.sentAtBlockHeight !== undefined && message.sentAtBlockHeight !== null) {
              return message.sentAtBlockHeight; // Use proper block height if available
          } else if (message.timestamp) {
              // For historical sent messages without sentAtBlockHeight, use timestamp for relative ordering
              // Assume 1 minute per block, estimate how many blocks ago this message was sent
              const now = Date.now();
              const messageTime = message.timestamp;
              const minutesAgo = Math.floor((now - messageTime) / 60000);
              
              // Don't go before block 0 and cap at a reasonable value
              // Messages more than currentBh blocks old will appear at the top
              return Math.max(0, currentBh - Math.min(currentBh, minutesAgo));
          }
          return null; // Last resort if no timestamp either
      } else { // Received
          if (message.confirmations === undefined || message.confirmations === null) {
              return null; // Cannot determine block if confirmations are missing
          }
          // Use confirmations for received. Handle unconfirmed (0 confs) as current block.
          return currentBh - message.confirmations;
      }
  }

</script>

<div class="flex flex-col h-screen bg-gray-50 font-sans text-sm">
  <!-- Top Bar -->
  <TopBar 
    verusIdName={loggedInIdentity?.formatted_name || 'Unknown User'} 
    privateBalance={privateBalance}
    blockHeight={blockHeight}
    isTransactionPending={isTransactionPending}
    on:logout={handleLogout}
    on:settings={handleSettings}
  />

  <!-- Main Content Area (Sidebar + Chat View / Settings View) -->
  <div class="flex flex-grow overflow-hidden border-t border-gray-200"> 
    
    <!-- Left Sidebar -->
    <div class="w-[25%] max-w-[300px] min-w-[200px] flex-shrink-0 bg-white border-r border-gray-200 flex flex-col shadow-sm">
      <ConversationsList 
        conversations={conversations}
        selectedConversationId={selectedConversationId}
        on:selectConversation={handleSelectConversation}
        on:openNewChatModal={handleOpenNewChatModal}  
      />
    </div>

    <!-- Right Panel (Chat View OR Settings View) -->
    <div class="flex-grow flex flex-col bg-gray-50">
      {#if showSettingsView}
          <SettingsView 
              currentPersistenceSetting={persistenceSetting}
              {loggedInUserIAddress}
              on:togglePersistence={handlePersistenceToggle}
              on:deleteHistory={handleDeleteHistory}
              on:closeSettings={handleCloseSettings}
          />
      {:else}
          <ConversationView 
              contactName={selectedContactName}
              messages={selectedMessages}
              privateBalance={privateBalance}
              isTransactionPending={isTransactionPending}
              on:sendMessage={handleSendMessage}
          />
      {/if}
    </div>
  </div>

  <!-- New Chat Modal -->
  <NewChatModal 
    bind:showModal={showNewChatModal}
    loggedInUserPrivateAddress={loggedInUserPrivateAddress}
    existingConversationIds={existingConversationIds}
    on:close={handleCloseNewChatModal}
    on:start-chat={handleStartChat}
  />

  <!-- Persistence Prompt Modal -->
  <PersistencePromptModal 
    bind:showModal={showPersistencePrompt}
    verusIdName={loggedInIdentity?.formatted_name || 'this VerusID'}
    on:save={() => handleSavePreference(true)}
    on:dontSave={() => handleSavePreference(false)}
  />

</div> 