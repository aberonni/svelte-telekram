<script lang="ts">
  import { Route, navigate as goto } from "svelte-navigator";
  import { createKaiNavigator } from '../utils/navigation';
  import { ListView, LoadingBar, Button, TextInputField, Toast, Toaster, SoftwareKey, TextInputDialog, OptionMenu } from '../components';
  import { onMount, onDestroy } from 'svelte';

  import { TelegramKeyHash, Api, AuthKey, client, session, cachedDatabase } from '../utils/bootstrap';

  import QRModal from '../widgets/QRModal.svelte';
  import ChatListView from '../widgets/ChatListView.svelte';
  import ArchivedChats from '../widgets/ArchivedChats.svelte';
  import ContactList from '../widgets/ContactList.svelte';

  import { connect, connectionStatus, authorizedStatus, isUserAuthorized, authorizedUser, chatCollections, retrieveChats, cachedThumbnails, dispatchMessageToClient, dispatchMessageToWorker } from '../stores/telegram';

  const navClass: string = 'homeNav';

  export let location: any;
  export let navigate: any;
  export let getAppProp: Function;

  let loadingBar: LoadingBar;
  let inputSoftwareKey: SoftwareKey;
  let qrModal: QRModal;
  let password2FA: TextInputDialog;
  let authorizedMenu: OptionMenu;
  let archivedChatListMenu: ArchivedChats;
  let contactListMenu: ContactList;

  let unchatCollections;
  let unauthorizedStatus;
  let unconnectionStatus;
  let uncachedThumbnails;
  let unauthorizedUser;

  let webWorkerStatus: bool = false;
  let name: string = 'Telekram';
  let phoneNumber = '';
  let phoneCode = '';
  let phoneCodeHash = null;
  let qrCode = null;
  let authStatus: boolean = false;
  let archivedChatList = [];
  let archivedChatListName = [];
  let user = [];

  let chatList = [];
  let thumbs = {};

  let navOptions = {
    verticalNavClass: navClass,
    softkeyLeftListener: function(evt) {
      if (inputSoftwareKey || qrModal|| password2FA)
        return;
      if (authStatus)
        openAuthorizedMenu();
    },
    softkeyRightListener: function(evt) {
      if (inputSoftwareKey || qrModal|| password2FA)
        return;
    },
    enterListener: function(evt) {
      if (inputSoftwareKey || qrModal|| password2FA)
        return;
      const navClasses = document.getElementsByClassName(navClass);
      if (navClasses[this.verticalNavIndex] != null) {
        navClasses[this.verticalNavIndex].click();
      }
    },
    backspaceListener: function(evt) {},
    arrowLeftListener: function(evt) {},
    arrowRightListener: function(evt) {},
  };

  let navInstance = createKaiNavigator(navOptions);

  function openArchivedChatListMenu() {
    archivedChatListMenu = new ArchivedChats({
      target: document.body,
      props: {
        title: 'Archived Chats',
        focusIndex: 0,
        options: archivedChatList,
        thumbs: thumbs,
        softKeyLeftText: 'Unarchive',
        softKeyCenterText: 'Select',
        softKeyRightText: '',
        onSoftkeyRight: (evt, scope) => {
          archivedChatListMenu.$destroy();
        },
        onSoftkeyLeft: (evt, scope) => {
          archivedChatListMenu.$destroy();
        },
        onEnter: (evt, scope) => {
          archivedChatListMenu.$destroy();
          setTimeout(() => {
            openRoom(scope.selected.name, scope.selected);
          }, 100);
        },
        onBackspace: (evt, scope) => {
          evt.preventDefault();
          evt.stopPropagation();
          archivedChatListMenu.$destroy();
        },
        onOpened: () => {
          navInstance.detachListener();
        },
        onClosed: (scope) => {
          navInstance.attachListener();
          archivedChatListMenu = null;
        }
      }
    });
  }

  async function getContacts() {
    try {
      const result = await client.invoke(
        new Api.contacts.GetContacts()
      );
      const sorted = result.users.sort((a, b) => {
        return [a.firstName || '', a.lastName || '', a.phone || ''].join(' ').localeCompare([b.firstName || '', b.lastName || '', b.phone || ''].join(' '));
      });
      contactListMenu = new ContactList({
        target: document.body,
        props: {
          title: 'Contact List',
          focusIndex: 0,
          sources: sorted,
          thumbs: thumbs,
          softKeyLeftText: 'Search',
          softKeyCenterText: 'Chat',
          softKeyRightText: '',
          onEnter: (evt, scope) => {
            contactListMenu.$destroy();
            setTimeout(async () => {
              if (scope.selected) {
                let chat = [...chatList, ...archivedChatList].find(c => scope.selected.id.value.toString() === c.id.value.toString());
                if (chat != null) {
                  openRoom(chat.name, chat);
                } else {
                  let name = '';
                  if (scope.selected.firstName)
                    name = scope.selected.firstName;
                  if (scope.selected.lastName)
                    name += ' ' + scope.selected.lastName;
                  if (name === '' && scope.selected.username)
                    name = scope.selected.username;
                  else if (name === '' && scope.selected.phone)
                    name = scope.selected.phone;
                  else if (name === '')
                    name = scope.selected.id.value.toString();
                  openRoom(name, { entity: scope.selected });
                }
              }
            }, 100);
          },
          onBackspace: (evt, scope) => {
            evt.preventDefault();
            evt.stopPropagation();
            contactListMenu.$destroy();
          },
          onOpened: () => {
            navInstance.detachListener();
          },
          onClosed: (scope) => {
            navInstance.attachListener();
            contactListMenu = null;
          }
        }
      });
    } catch (err) {
      console.log(err);
    }
  }

  async function getSavedContacts() {
    try {
      let result = await client.invoke(new Api.contacts.GetSaved());
      console.log(result);
    } catch (err) {
      if (err.errorMessage === "TAKEOUT_REQUIRED") {
        getContactsFallback();
      }
    }
  }

  async function getSavedContactsFallback() {
    try {
      let result = await client.invoke(new Api.account.InitTakeoutSession({ contacts: true }));
      result = await client.invoke(new Api.InvokeWithTakeout({ takeoutId: result.id, query: new Api.contacts.GetSaved() }));
      console.log(result);
    } catch (err) {
      console.log(err);
    }
  }

  async function openAuthorizedMenu() {
    authorizedMenu = new OptionMenu({
      target: document.body,
      props: {
        title: user.length > 0 ? `Hi ${user[0].username}` : 'Menu',
        focusIndex: 0,
        options: [
          { title: 'New Contact' },
          { title: 'Contacts' },
          { title: 'Settings' },
          { title: 'Logout' },
          { title: 'Exit' },
        ],
        softKeyCenterText: 'select',
        onSoftkeyRight: (evt, scope) => {},
        onSoftkeyLeft: (evt, scope) => {},
        onEnter: async(evt, scope) => {
          authorizedMenu.$destroy();
          if (scope.selected.title === 'Contacts') {
            getContacts();
          } else if (scope.selected.title === 'Logout') {
            signOut();
          } else if (scope.selected.title === 'Exit') {
            window.close();
          }
        },
        onBackspace: (evt, scope) => {
          evt.preventDefault();
          evt.stopPropagation();
          authorizedMenu.$destroy();
        },
        onOpened: () => {
          navInstance.detachListener();
        },
        onClosed: (scope) => {
          navInstance.attachListener();
          authorizedMenu = null;
        }
      }
    });
  }

  function resetCursor() {
    if (qrModal != null || password2FA != null)
      return
    navInstance.verticalNavIndex = 0;
    setTimeout(() => {
      navInstance.navigateListNav(0);
      setTimeout(() => {
        const cursor = document.getElementsByClassName(navClass)[navInstance.verticalNavIndex];
        if (cursor) {
          cursor.classList.add('focus');
        }
      }, 150)
    }, 150);
  }

  function toastMessage(text = 'I\'m out after 2 second') {
    const t = new Toast({
      target: document.body,
      props: {
        options: {}
      }
    })
    Toaster.push(text , {
      dismissable: false,
      intro: { y: -64 },
      duration: 2000,
      onpop: () => {
        setTimeout(() => {
          t.$destroy();
        }, 4000);
      }
    })
  }

  function showLoadingBar() {
    loadingBar = new LoadingBar({
      target: document.body,
      props: {
        onOpened: () => {
          navInstance.detachListener();
        },
        onClosed: () => {
          navInstance.attachListener();
          loadingBar = null;
        }
      }
    });
  }

  function onInputPhoneNumber(evt) {
    phoneNumber = evt.target.value.trim().toString();
  }

  function onInputPhoneCode(evt) {
    phoneCode = evt.target.value.trim().toString();
  }

  function onFocus(evt) {
    if (qrModal != null || password2FA != null)
      return
    inputSoftwareKey = new SoftwareKey({
      target: document.body,
      props: {
        isInvert: true,
        leftText: '',
        centerText: 'Enter',
        rightText: ''
      }
    });
  }

  function onBlur(evt) {
    if (inputSoftwareKey) {
      inputSoftwareKey.$destroy();
      inputSoftwareKey = null;
    }
  }

  function sign_up() {}

  // EVENT 2
  function sendCode() {
    if (!webWorkerStatus)
      return;
    const params = {
      type: 2,
      params: {
        phoneNumber: phoneNumber,
        apiId: TelegramKeyHash.api_id,
        apiHash: TelegramKeyHash.api_hash,
        settings: {
          allowFlashcall: true,
          currentNumber: true,
          allowAppHash: true,
        },
      }
    }
    if (loadingBar == null)
      showLoadingBar();
    dispatchMessageToWorker.emit('message', params);
  }

  // EVENT 3
  function signIn() {
    if (!webWorkerStatus)
      return;
    const params = {
      type: 3,
      params: {
        phoneNumber: phoneNumber,
        phoneCodeHash: phoneCodeHash,
        phoneCode: phoneCode,
      }
    }
    if (loadingBar == null)
      showLoadingBar();
    dispatchMessageToWorker.emit('message', params);
  }

  // EVENT 4
  function signIn2FA() {
    if (!webWorkerStatus)
      return;
    password2FA = new TextInputDialog({
      target: document.body,
      props: {
        title: '2FA Password',
        softKeyCenterText: 'ok',
        softKeyRightText: 'Cancel',
        value: '',
        placeholder: 'Password',
        type: 'text',
        onSoftkeyLeft: (evt, value) => {},
        onSoftkeyRight: (evt, value) => {
          password2FA.$destroy();
        },
        onEnter: (evt, password) => {
          const params = {
            type: 4,
            params: {
              apiId: TelegramKeyHash.api_id,
              apiHash: TelegramKeyHash.api_hash,
              password: password
            }
          }
          if (loadingBar == null)
            showLoadingBar();
          dispatchMessageToWorker.emit('message', params);
        },
        onBackspace: (evt, value) => {
          evt.stopPropagation();
        },
        onOpened: () => {
          navInstance.detachListener();
        },
        onClosed: (value) => {
          navInstance.attachListener();
          password2FA = null;
        }
      }
    });
  }

  // EVENT 5
  function signInQR() {
    if (!webWorkerStatus)
      return;
    setTimeout(() => {
      qrModal = new QRModal({
        target: document.body,
        props: {
          title: 'Log-In via QR Code',
          onBackspace: (evt) => {
            evt.preventDefault();
            evt.stopPropagation();
            qrModal.$destroy();
          },
          onOpened: () => {
            navInstance.detachListener();
          },
          onClosed: () => {
            navInstance.attachListener();
            qrModal = null;
          }
        }
      });
    }, 100);
  }

  // EVENT 6
  function exportLoginToken() {
    if (!webWorkerStatus)
      return;
    const params = {
      type: 6,
      params: {
        apiId: TelegramKeyHash.api_id,
        apiHash: TelegramKeyHash.api_hash,
        exceptIds: []
      }
    }
    if (loadingBar == null)
      showLoadingBar();
    dispatchMessageToWorker.emit('message', params);
  }

  // EVENT 7
  function importLoginToken(token) {
    if (!webWorkerStatus)
      return;
    const params = {
      type: 7,
      params: {
        token: token
      }
    }
    if (loadingBar == null)
      showLoadingBar();
    dispatchMessageToWorker.emit('message', params);
  }

  function resetSignIn() {
    phoneCodeHash = null;
    resetCursor();
  }

  async function signOut() {
    try {
      const result = await client.invoke(new Api.auth.LogOut({}));
      isUserAuthorized();
      phoneCodeHash = null;
    } catch (err) {
      console.log(err);
    }
  }

  function sortChats(chats) {
    try {
      archivedChatListName = [];
      archivedChatList = [];
      let tempChatList = [];
      chats.forEach((chat, index) => {
        if (chat.archived) {
          if (archivedChatListName.length !== 3) {
            archivedChatListName.push(chat.name);
            if (archivedChatListName.length === 2)
              archivedChatListName.push('more...')
          }
          archivedChatList.push(chat);
        } else {
          tempChatList.push(chat);
        }
      });
      chatList = tempChatList;
      //setTimeout(() => {
        //navInstance.navigateListNav(1);
        //setTimeout(() => {
          //navInstance.navigateListNav(-1);
        //}, 100);
        //console.log(1);
      //}, 100);
    } catch(err) {
      console.log(err.toString());
    }
  }

  async function openRoom(name, chat) {
    let scrollAt;
    try {
      let pref = await (await cachedDatabase).get('chatPreferences', chat.entity.id.value.toString());
      scrollAt = pref['scrollAt'];
    } catch(err) {}
    goto('room', { state: { name, entity: chat.entity.toJSON(), scrollAt } });

  }

  function eventHandler(evt) {
    if (evt.className === "UpdateLoginToken") {
      exportLoginToken();
      if (qrModal) {
        qrModal.$destroy();
      }
    }
  }

  function getThumb(chat) {
    if (chat.entity.__isSavedMessages) {
      return `<img alt="icon" style="background-color:var(--themeColor);width:40px;height:40px;border-radius:50%;box-sizing:border-box;border: 2px solid #fff;"" src="/icons/saved.png"/>`
    }
    if (thumbs[chat.iconRef]) {
      return `<img alt="icon" style="background-color:var(--themeColor);width:40px;height:40px;border-radius:50%;box-sizing:border-box;border: 2px solid #fff;"" src="${thumbs[chat.iconRef]}"/>`;
    }
    return `<div style="display:flex;flex-direction:column;justify-content:center;align-items:center;font-weight:bold;color:#fff;background-color:var(--themeColor);width:40px;height:40px;border-radius:50%;box-sizing:border-box;border: 2px solid #fff;">${chat.name.split(' ').map(text => text[0]).splice(0, 2).join('')}</div>`;
  }

  async function handleWebWorkerMessage(data: any) {
    try {
      // console.log('dispatchMessageToClient:', data.type);
      switch (data.type) {
        case -1:
          console.error('Error', data.params);
          break;
        case 0:
          console.log('Connected to authenticationWebWorker');
          break;
        case 1:
          console.log('authenticationWebWorker.client.event:', data.params);
          if (data.params.state && data.params.state === 1)
            webWorkerStatus = true;
          else if (data.params.state && data.params.state === -1)
            webWorkerStatus = false;
          if (data.params.data && data.params.data.className === "UpdateLoginToken") {
            exportLoginToken();
            if (qrModal) {
              qrModal.$destroy();
            }
          }
          break;
        case 2:
          if (loadingBar)
            loadingBar.$destroy();
          console.log('sendCode:', data.params.phoneCodeHash);
          phoneCodeHash = data.params.phoneCodeHash;
          resetCursor();
          break;
        case -2:
          if (loadingBar)
            loadingBar.$destroy();
          console.error('sendCode:', data.params);
          toastMessage(data.params || "ERROR SEND CODE");
          break;
        case 3:
          if (loadingBar)
            loadingBar.$destroy();
          console.log('signIn:', data.params.session.authKey, data.params.session.dcId);
          resetCursor();
          phoneCodeHash = null;
          if (data.params.session) {
            const authKey = new AuthKey(data.params.session.authKey._key, data.params.session.authKey._hash);
            session.setDC(data.params.session.dcId, data.params.session.serverAddress, data.params.session.port);
            session.setAuthKey(authKey);
            session.authKey = authKey;
            const typedArray = authKey.getKey();
            const arr = Array.from ? Array.from(typedArray) : [].map.call(typedArray, (v => v));
            const str = JSON.stringify(arr);
            window.localStorage.setItem('gramjs:authKey', `{ "type": "Buffer", "data": ${str} }`);
            alert('Please re-launch the app');
            window.close();
          }
          break;
        case -3:
          if (loadingBar)
            loadingBar.$destroy();
          console.error('signIn:', data.params);
          if (data.params !== 'SESSION_PASSWORD_NEEDED') {
            console.error('signIn:', data.params);
            toastMessage(data.params || "ERROR SIGN IN");
            return;
          }
          signIn2FA();
          break;
        case 4:
          if (loadingBar)
            loadingBar.$destroy();
          console.log('signIn2FA:', data.params);
          if (password2FA) {
            password2FA.$destroy();
          }
          resetCursor();
          phoneCodeHash = null;
          if (data.params.session) {
            const authKey = new AuthKey(data.params.session.authKey._key, data.params.session.authKey._hash);
            session.setDC(data.params.session.dcId, data.params.session.serverAddress, data.params.session.port);
            session.setAuthKey(authKey);
            session.authKey = authKey;
            const typedArray = authKey.getKey();
            const arr = Array.from ? Array.from(typedArray) : [].map.call(typedArray, (v => v));
            const str = JSON.stringify(arr);
            window.localStorage.setItem('gramjs:authKey', `{ "type": "Buffer", "data": ${str} }`);
            alert('Please re-launch the app');
            window.close();
          }
          break;
        case -4:
          if (loadingBar)
            loadingBar.$destroy();
          console.error('signIn2FA:', data.params);
          toastMessage(data.params || "ERROR SIGN IN 2FA");
          break;
        case 6:
          if (loadingBar)
            loadingBar.$destroy();
          console.log('exportLoginToken:', data.params);
          if (data.params.result.className.toLocaleLowerCase() === 'auth.LoginTokenSuccess'.toLocaleLowerCase()) {
            resetCursor();
            phoneCodeHash = null;
            if (data.params.session) {
              const authKey = new AuthKey(data.params.session.authKey._key, data.params.session.authKey._hash);
              session.setDC(data.params.session.dcId, data.params.session.serverAddress, data.params.session.port);
              session.setAuthKey(authKey);
              session.authKey = authKey;
              const typedArray = authKey.getKey();
              const arr = Array.from ? Array.from(typedArray) : [].map.call(typedArray, (v => v));
              const str = JSON.stringify(arr);
              window.localStorage.setItem('gramjs:authKey', `{ "type": "Buffer", "data": ${str} }`);
              alert('Please re-launch the app');
              window.close();
            }
          } else if (['auth.logintoken', 'auth.logintokensuccess', 'auth.logintokenmigrateto'].indexOf(data.params.result.className.toLocaleLowerCase() > -1)) {
            importLoginToken(data.params.result.token);
          } else {
            console.error('importLoginToken', data.params.result);
            toastMessage(data.params.result.className || "ERROR IMPORT LOGIN TOKEN");
          }
          break;
        case -6:
          if (loadingBar)
            loadingBar.$destroy();
          console.log('exportLoginToken:', data.params);
          if (data.params !== 'SESSION_PASSWORD_NEEDED') {
            console.error('exportLoginToken:', data.params);
            toastMessage(data.params || "ERROR EXPORT LOGIN TOKEN");
            return;
          }
          signIn2FA();
          break;
        case 7:
          console.log('importLoginToken:', data.params);
          resetCursor();
          phoneCodeHash = null;
          if (data.params.session) {
            const authKey = new AuthKey(data.params.session.authKey._key, data.params.session.authKey._hash);
            session.setDC(data.params.session.dcId, data.params.session.serverAddress, data.params.session.port);
            session.setAuthKey(authKey);
            session.authKey = authKey;
            const typedArray = authKey.getKey();
            const arr = Array.from ? Array.from(typedArray) : [].map.call(typedArray, (v => v));
            const str = JSON.stringify(arr);
            window.localStorage.setItem('gramjs:authKey', `{ "type": "Buffer", "data": ${str} }`);
            alert('Please re-launch the app');
            window.close();
          }
          break;
        case -7:
          if (loadingBar)
            loadingBar.$destroy();
          console.error('importLoginToken:', data.params);
          toastMessage(data.params || "ERROR IMPORT LOGIN TOKEN");
          break;
      }
    } catch (err) {
      console.error('dispatchMessageToClient:', err);
    }
  }

  onMount(() => {
    const { appBar, softwareKey } = getAppProp();
    appBar.setTitleText(name);
    softwareKey.setText({ left: '', center: 'SELECT', right: '' });
    navInstance.attachListener();

    // client.addEventHandler(eventHandler);

    unchatCollections = chatCollections.subscribe(chats => {
      if (client.connected) {
        sortChats(chats);
      }
    });

    unconnectionStatus = connectionStatus.subscribe(status => {
      // console.log(status);
    });

    unauthorizedStatus = authorizedStatus.subscribe(status => {
      authStatus = status;
      if (status) {
        softwareKey.setLeftText('Menu');
        softwareKey.setRightText('Search');
        if (inputSoftwareKey) {
          inputSoftwareKey.$destroy();
          inputSoftwareKey = null;
        }
        if (qrModal) {
          qrModal.$destroy();
          qrModal = null;
        }
        if (password2FA) {
          password2FA.$destroy();
          password2FA = null;
        }
      } else {
        softwareKey.setLeftText('');
        softwareKey.setRightText('');
      }
    });

    uncachedThumbnails = cachedThumbnails.subscribe(data => {
      thumbs = data;
      chatList = [...chatList];
    });

    unauthorizedUser = authorizedUser.subscribe(data => {
      user = data;
    });

    dispatchMessageToClient.addListener('message', handleWebWorkerMessage);

  });

  onDestroy(() => {
    client.removeEventHandler(eventHandler);
    navInstance.detachListener();
    if (unchatCollections)
      unchatCollections();
    if (unauthorizedStatus)
      unauthorizedStatus();
    if (unconnectionStatus)
      unconnectionStatus();
    if (uncachedThumbnails)
      uncachedThumbnails();
    if (unauthorizedUser)
      unauthorizedUser();
    dispatchMessageToClient.removeListener('message', handleWebWorkerMessage);
  });

</script>

<main id="home-screen" data-pad-top="28" data-pad-bottom="30">
  {#if authStatus === false }
  {#if phoneCodeHash === null}
  <TextInputField className="{navClass}" label="Phone Number" placeholder="{webWorkerStatus ? 'Phone Number' : 'Connecting...'}" value={phoneNumber} type="tel" onInput="{onInputPhoneNumber}" {onFocus} {onBlur} />
  <Button className="{navClass}" text="Send Code" onClick={sendCode}>
    <span slot="leftWidget" class="kai-icon-arrow" style="margin:0px 5px;-moz-transform: scale(-1, 1);-webkit-transform: scale(-1, 1);-o-transform: scale(-1, 1);-ms-transform: scale(-1, 1);transform: scale(-1, 1);"></span>
    <span slot="rightWidget" class="kai-icon-arrow" style="margin:0px 5px;"></span>
  </Button>
  <Button className="{navClass}" text="Log-In via QR Code" onClick={signInQR}>
    <span slot="leftWidget" class="kai-icon-arrow" style="margin:0px 5px;-moz-transform: scale(-1, 1);-webkit-transform: scale(-1, 1);-o-transform: scale(-1, 1);-ms-transform: scale(-1, 1);transform: scale(-1, 1);"></span>
    <span slot="rightWidget" class="kai-icon-arrow" style="margin:0px 5px;"></span>
  </Button>
  {:else}
  <TextInputField className="{navClass}" label="Login Code" placeholder="Login Code" value={phoneCode} type="tel" onInput="{onInputPhoneCode}" {onFocus} {onBlur} />
  <Button className="{navClass}" text="Sign In" onClick={signIn}>
    <span slot="leftWidget" class="kai-icon-arrow" style="margin:0px 5px;-moz-transform: scale(-1, 1);-webkit-transform: scale(-1, 1);-o-transform: scale(-1, 1);-ms-transform: scale(-1, 1);transform: scale(-1, 1);"></span>
    <span slot="rightWidget" class="kai-icon-arrow" style="margin:0px 5px;"></span>
  </Button>
  <Button className="{navClass}" text="Return" onClick={resetSignIn}>
    <span slot="leftWidget" class="kai-icon-arrow" style="margin:0px 5px;-moz-transform: scale(-1, 1);-webkit-transform: scale(-1, 1);-o-transform: scale(-1, 1);-ms-transform: scale(-1, 1);transform: scale(-1, 1);"></span>
    <span slot="rightWidget" class="kai-icon-arrow" style="margin:0px 5px;"></span>
  </Button>
  {/if}
  <Button className="{navClass}" text="Exit" onClick={() => window.close() }>
    <span slot="leftWidget" class="kai-icon-arrow" style="margin:0px 5px;-moz-transform: scale(-1, 1);-webkit-transform: scale(-1, 1);-o-transform: scale(-1, 1);-ms-transform: scale(-1, 1);transform: scale(-1, 1);"></span>
    <span slot="rightWidget" class="kai-icon-arrow" style="margin:0px 5px;"></span>
  </Button>
  {:else}
  {#if archivedChatList.length > 0 }
    <ListView className="{navClass}" title="Archived Chats" subtitle="{archivedChatListName.join(', ').substring(0, 50)}" onClick={openArchivedChatListMenu}>
      <span slot="leftWidget" style="padding-right: 4px;"><img alt="icon" style="background-color:var(--themeColor);width:40px;height:40px;box-sizing:border-box;border-radius:50%;border: 2px solid #fff;" src="/icons/archived.png"/></span>
    </ListView>
  {/if}
  {#each chatList as chat}
    <ChatListView userId={user.length > 0 ? user[0].id.value.toString() : null} chat={chat} className="{navClass}" icon={getThumb(chat)} onClick={() => openRoom(chat.name, chat)} />
  {/each}
  {/if}
</main>

<style>
  #home-screen {
    overflow: scroll;
    overflow-x: hidden;
    width: 100%;
  }
</style>
