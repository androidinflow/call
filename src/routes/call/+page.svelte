<script>
    import { onMount } from 'svelte';
    import { browser } from '$app/environment';
    
    let localStream;
    let peer;
    let peerId = '';
    let remotePeerId = '';
    let status = 'disconnected';
    
    onMount(async () => {
        if (browser) {
            const { Peer } = await import('peerjs');
            peer = new Peer();
            
            // Get local stream
            try {
                localStream = await navigator.mediaDevices.getUserMedia({ audio: true });
            } catch (err) {
                console.error('Failed to get local stream', err);
            }
            
            // When peer is ready
            peer.on('open', (id) => {
                peerId = id;
                status = 'ready';
            });
            
            // Handle incoming calls
            peer.on('call', (call) => {
                status = 'receiving call';
                
                // Answer the call with our local stream
                call.answer(localStream);
                
                // Handle receiving remote stream
                call.on('stream', (remoteStream) => {
                    const remoteAudio = document.getElementById('remoteAudio');
                    remoteAudio.srcObject = remoteStream;
                    status = 'connected';
                });
            });
        }
    });
    
    // Function to make a call
    async function makeCall() {
        if (!remotePeerId) return;
        
        status = 'calling';
        const call = peer.call(remotePeerId, localStream);
        
        call.on('stream', (remoteStream) => {
            const remoteAudio = document.getElementById('remoteAudio');
            remoteAudio.srcObject = remoteStream;
            status = 'connected';
        });
    }
</script>

<div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Audio Call App</h1>
    
    <div class="mb-4">
        <p>Your Peer ID: {peerId}</p>
        <p>Status: {status}</p>
    </div>
    
    <div class="mb-4">
        <input
            type="text"
            bind:value={remotePeerId}
            placeholder="Enter remote peer ID"
            class="border p-2 rounded"
        />
        <button
            on:click={makeCall}
            class="bg-blue-500 text-white px-4 py-2 rounded ml-2"
            disabled={!remotePeerId || status === 'connected'}
        >
            Call
        </button>
    </div>
    
    <audio id="remoteAudio" autoplay></audio>
</div>
