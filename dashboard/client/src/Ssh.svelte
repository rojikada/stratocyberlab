<script>
    import {onDestroy, onMount} from 'svelte';
    import { createEventDispatcher } from 'svelte'

    import {Terminal} from '@xterm/xterm'
    import {FitAddon} from '@xterm/addon-fit';
    import '@xterm/xterm/css/xterm.css';
    import {io} from 'socket.io-client';

    const dispatch = createEventDispatcher()

    let terminalContainer;
    let terminal;
    let ws;

    export let resize;

    let isTerminalFocused = false;

    function handleCtrlW(event) {
        if (event.ctrlKey && event.key === 'w' && isTerminalFocused) {
            event.preventDefault(); // Prevent browser tab close
            terminal.write('\x17'); // propagate it to terminal
        }
    }

    function addListenersToHandleCtrlW() {
        terminalContainer.addEventListener('focus', () => {
            isTerminalFocused = true;
        });

        terminalContainer.addEventListener('blur', () => {
            isTerminalFocused = false;
        });

        window.addEventListener('keydown', handleCtrlW);
    }

    function deleteListenersToHandleCtrlW() {
        window.removeEventListener('keydown', handleCtrlW);
        terminalContainer.removeEventListener('focus', () => {
            isTerminalFocused = true;
        });
        terminalContainer.removeEventListener('blur', () => {
            isTerminalFocused = false;
        });
    }

    onMount(() => {
        terminal = new Terminal({
            cursorBlink: true,
        });
        const fitAddon = new FitAddon();
        terminal.loadAddon(fitAddon);
        terminal.open(terminalContainer);
        fitAddon.fit();

        function emitTerminalSize() {
            socket.emit('ssh_resize', {
                "cols": terminal.cols,
                "rows": terminal.rows
            });
        }

        resize = function() {
            fitAddon.fit()
            emitTerminalSize()
        }

        // Whenever resize happens, we dynamically resize size of terminal
        // and send the information to the server as-well
        window.addEventListener('resize', () => {
            resize()
        });

        // Prevent browser's Ctrl+W from closing the tab
        window.addEventListener('keydown', handleCtrlW);

        const socket = io(`ws://${window.location.host}:8080/`, {
            path: '/socket.io',
            transports: ['websocket']
        });

        socket.on('connect', () => {
            emitTerminalSize()
        });

        socket.on('ssh_output', (data) => {
            const decoder = new TextDecoder('utf-8');
            const text = decoder.decode(data);
            terminal.write(text);
        });

        terminal.onData(data => {
            socket.emit('ssh_input', data);
        });

        addListenersToHandleCtrlW()
    });


  onDestroy(() => {
    if (terminal) {
      terminal.dispose();
    }
    if (ws) {
      ws.close();
    }

    deleteListenersToHandleCtrlW()
  });

  function hide(){
      dispatch("hide")
  }

</script>

<style>
    .close-button {
        position: absolute;
        right: 10px;
        top: 0;
        opacity: 0.5;
        transition: opacity 0.3s ease;
        border-radius: 0 0 5px 5px;
    }

    .close-button:hover {
        opacity: 1;
    }
</style>


<div style="position: relative;" class="h-100">
    <div bind:this={terminalContainer} class="h-100"></div>
    <button class="btn btn-secondary close-button" on:click={hide}>
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <line x1="6" y1="12" x2="18" y2="12" stroke="white" stroke-width="2" stroke-linecap="round"/>
        </svg>
    </button>
</div>