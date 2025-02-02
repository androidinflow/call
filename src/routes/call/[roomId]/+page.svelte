<script>
    import { onMount } from 'svelte';
    import { browser } from '$app/environment';
    import { page } from '$app/stores';
    
    let localStream;
    let peer;
    let status = 'disconnected';
    let error = '';
    
    onMount(async () => {
        if (browser) {
            const { Peer } = await import('peerjs');
            
            // Get room ID from URL parameters
            const roomId = $page.params.roomId;
            // Get user IDs from query parameters
            const urlParams = new URLSearchParams(window.location.search);
            const userId = urlParams.get('userId');
            const connectTo = urlParams.get('connect');
            
            console.log('Room:', roomId, 'User:', userId, 'Connect to:', connectTo);
            
            if (!userId) {
                error = 'No user ID provided';
                return;
            }
            
            // Initialize peer with userId
            peer = new Peer(`${roomId}_${userId}`, {
                config: {
                    iceServers: [
                        { urls: 'stun:stun.l.google.com:19302' },
                        { urls: 'stun:stun1.l.google.com:19302' },
                        { urls: 'stun:stun2.l.google.com:19302' }
                    ]
                },
                debug: 3
            });
            
            try {
                localStream = await navigator.mediaDevices.getUserMedia({ 
                    audio: true,
                    video: false
                });
            } catch (err) {
                error = 'میکروفون را فعال کنید';
                console.error('Failed to get local stream', err);
                return;
            }
            
            peer.on('open', (id) => {
                status = 'ready';
                console.log('Connected with ID:', id);
                
                // If there's a connect parameter, automatically initiate the call
                if (connectTo) {
                    makeCall(`${roomId}_${connectTo}`);
                }
            });
            
            peer.on('error', (err) => {
                console.error('PeerJS error:', err);
                error = `خطا در اتصال: ${err.type}`;
                status = 'error';
            });
            
            peer.on('call', (call) => {
                status = 'دریافت تماس';
                console.log('Receiving call from:', call.peer);
                
                call.answer(localStream);
                
                call.on('stream', (remoteStream) => {
                    const remoteAudio = document.getElementById('remoteAudio');
                    remoteAudio.srcObject = remoteStream;
                    status = 'متصل شد';
                });
            });
        }
    });
    
    async function makeCall(remotePeerId) {
        status = 'در حال تماس';
        console.log('Connecting to:', remotePeerId);
        
        try {
            const call = peer.call(remotePeerId, localStream);
            
            call.on('stream', (remoteStream) => {
                const remoteAudio = document.getElementById('remoteAudio');
                remoteAudio.srcObject = remoteStream;
                status = 'متصل شد';
            });
        } catch (err) {
            console.error('Make call error:', err);
            error = `خطا در برقراری تماس: ${err.message}`;
            status = 'error';
        }
    }
</script>

<div class="container mx-auto p-4 text-center">
    <h1 class="text-2xl font-bold mb-4">تماس صوتی</h1>
    
    {#if error}
        <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
            {error}
        </div>
    {/if}
    
    <div class="mb-4">
        <p class="text-lg">وضعیت: {status}</p>
    </div>
    
    <audio id="remoteAudio" autoplay></audio>
</div>
