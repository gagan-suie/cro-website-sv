<script lang="ts">
	import DrawerChat from '$lib/components/Channel/Chat/DrawerChat.svelte'
	import StreamContainer from '$lib/components/Channel/Stream/StreamContainer.svelte'
	import { onDestroy, onMount } from 'svelte'
	import { get, del } from '$lib/api'
	import {
		emitChatHistoryToChannel,
		initChannelSocket,
		channelSocket,
		emitChannelSubscribeByUser,
		emitDeleteAllMessagesToChannel
	} from '$lib/websocket'
	import { channel_connection, channel_message } from '$lib/stores/websocketStore'
	import { isJsonString } from '$lib/utils'
	import { is_chat_drawer_open, is_chat_drawer_destroy } from '$lib/stores/channelStore'
	import Modal from '$lib/components/Global/Modal.svelte'
	import { goto } from '$app/navigation'
	import { page } from '$app/stores'
	import DrawerEditChannel from '$lib/components/Channel/Chat/DrawerEditChannel.svelte'
	import {
		is_sharing_screen,
		is_sharing_webcam,
		is_sharing_audio,
		updateVideoItems,
		video_items
	} from '$lib/stores/streamStore'

	let channel: any
	let channelId = $page.params.channelId || ''
	$: userCount = 0
	$: isHostOrGuest = isHost || channel?.guests?.includes($page.data.user?.userId)

	let isDeleteModalOpen = false,
		showEditChannelDrawer = false,
		channels: any = [],
		skip = 0,
		limit = 10,
		host = {},
		isHost = false

	$: if (channel) {
		if (channel._id !== channelId) {
			channelSocket.close()
			channelId = channel?._id
			$channel_message = ''
			$video_items = []
			if (isHostOrGuest) {
				$is_sharing_screen = false
				$is_sharing_webcam = false
				$is_sharing_audio = false
			} else {
				$is_sharing_screen = undefined
				$is_sharing_webcam = undefined
				$is_sharing_audio = undefined
			}
			handleWebsocket()
		}
	}

	onMount(async () => {
		channelId = $page.params.channelId || ''
		await loadChannel()
		await handleWebsocket()
		await loadMoreChannels()
		$is_chat_drawer_destroy = false
		setTimeout(() => {
			$is_chat_drawer_open = true
		}, 600)
	})

	onDestroy(() => {
		if (isHostOrGuest) {
			$is_sharing_screen = false
			$is_sharing_webcam = false
			$is_sharing_audio = false
		}
		channelSocket?.close()
	})

	const loadChannel = async () => {
		channel = await get(`channel?channelId=${channelId}`)
		await getHost(channel.user)
		channels.push(channel)
	}

	const getHost = async (userId: string) => {
		host = await get(`users/search/id?userId=${userId}`)
		isHost = userId === $page?.data?.user?.userId
	}

	const handleWebsocket = async () => {
		try {
			const channelSocketId = await get(`wsinit/channelid?channelId=${channelId}`)
			initChannelSocket(channelSocketId)
			channelSocket.addEventListener('open', async (data) => {
				console.log('channel socket connection open', channelSocketId)
				$channel_connection = 'open'
				$channel_message = ''
				$video_items = []
				emitChannelSubscribeByUser({
					channelId,
					userId: $page.data.user?.userId,
					username: $page.data.user?.user?.username
				})
				emitChatHistoryToChannel({ channelId, skip: 100 })
				getHost(channel.user)
			})
			channelSocket.addEventListener('message', (data) => {
				console.log('channel listening to messages')
				if (isJsonString(data.data)) {
					console.log('data.data', data.data)
					$channel_message = data.data
				}
			})
			channelSocket.addEventListener('error', (data) => {
				console.log('channel socket connection error')
				console.log(data)
			})
			channelSocket.addEventListener('close', (data) => {
				console.log('channel socket connection close')
				console.log(data)
				$channel_connection = 'close'
				$channel_message = ''
				$video_items = []

				console.log('got here----', data.code)
				//if manually closed, don't reconnect
				if (data.code === 1005) return
				attemptReconnect()
			})
		} catch (error) {
			if (error) attemptReconnect()
		}
	}

	const attemptReconnect = () => {
		setTimeout(async () => {
			console.log('Reconnecting to WebSocket...')
			await handleWebsocket()
		}, 4000)
	}

	const deleteChannelNoAction = () => {
		isDeleteModalOpen = false
	}

	const deleteChannelYesAction = async () => {
		await del(`channels?channelId=${channelId}&bucketName=attachments`, {
			userId: $page.data.user?.userId,
			token: $page.data.user?.token
		})
		emitDeleteAllMessagesToChannel({ channelId })
		goto('/browse')
	}

	const loadMoreChannels = async () => {
		let newchannels = await get(`channels?skip=${skip}&limit=${limit}`)
		channels = [...channels, ...newchannels]
		skip += limit
	}

	channel_message.subscribe(async (value: any) => {
		if (!value) return
		var parsedMsg = JSON.parse(value)
		switch (parsedMsg.eventName) {
			case `channel-update-${channelId}`:
				console.log('channel-update', parsedMsg)
				channel = parsedMsg.channel
				break
			case `channel-subscribe-${channelId}`:
				userCount = parsedMsg.userCount
				if (parsedMsg.quitUserId) {
					$video_items = $video_items.filter((video: any) => video._id !== parsedMsg.quitUserId)
				} else {
					const activeGuests = parsedMsg.activeGuests
					if (activeGuests?.length) {
						if ($video_items.length) {
							// for users that are in the channel and new users join
							// add new users but dont overwrite the existing ones streaming
							$video_items = activeGuests.map((guest: any) => {
								const foundVideoItem = $video_items.find((video: any) => guest._id === video._id)
								return foundVideoItem || guest
							})
						} else {
							// for new users joining the channel
							const liveInputs = await get(`cloudflare/live-inputs?channelId=${channelId}`)
							$video_items = updateVideoItems([...activeGuests], liveInputs)
						}
					}
				}
				break
			case `channel-streaming-action-${channelId}`:
				switch (parsedMsg.data.action) {
					case 'toggleTrack':
						if ($page.data.user?.userId) {
							if ($page.data.user.userId !== parsedMsg.data.video._id) {
								$video_items = updateVideoItems($video_items, [parsedMsg.data.video])
							}
						} else {
							$video_items = updateVideoItems($video_items, [parsedMsg.data.video])
						}
						break
				}
				break
		}
	})
</script>

{#if channel && channel._id === channelId}
	<div class="flex flex-auto">
		<div class="drawer drawer-end">
			<input
				id="chat-drawer"
				type="checkbox"
				class="drawer-toggle"
				bind:checked={$is_chat_drawer_open} />
			<div class="drawer-content">
				<StreamContainer
					bind:channel
					bind:userCount
					bind:channels
					on:loadMore={loadMoreChannels}
					bind:isHostOrGuest />

				{#if showEditChannelDrawer}
					<DrawerEditChannel bind:channel bind:showDrawer={showEditChannelDrawer} />
				{/if}
			</div>
			{#if !$is_chat_drawer_destroy}
				<div
					class="drawer-side m-5 rounded-lg md:w-fit lg:drop-shadow-lg"
					class:!hidden={showEditChannelDrawer}>
					<label for="chat-drawer" class="drawer-overlay" />

					<DrawerChat bind:channel bind:showEditChannelDrawer bind:host bind:isHost />
				</div>
			{/if}
		</div>
	</div>

	<input
		type="checkbox"
		id="modal-delete-channel"
		class="modal-toggle"
		bind:checked={isDeleteModalOpen} />
	<Modal
		id="modal-delete-channel"
		title="Delete channel"
		message="Are you sure you want to delete this channel?"
		no="Cancel"
		noAction={deleteChannelNoAction}
		yes="Yes"
		yesAction={deleteChannelYesAction}
		isError={true} />
{/if}
