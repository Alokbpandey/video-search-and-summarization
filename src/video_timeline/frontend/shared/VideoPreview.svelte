<!--
 * SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
 * SPDX-License-Identifier: LicenseRef-NvidiaProprietary
 *
 * NVIDIA CORPORATION, its affiliates and licensors retain all intellectual
 * property and proprietary rights in and to this material, related
 * documentation and any modifications thereto. Any use, reproduction,
 * disclosure or distribution of this material and related documentation
 * without an express license agreement from NVIDIA CORPORATION or
 * its affiliates is strictly prohibited.
-->

<script lang="ts">
	import { createEventDispatcher, afterUpdate, tick } from "svelte";
	import {
		BlockLabel,
		Empty,
		IconButton,
		ShareButton,
		IconButtonWrapper
	} from "@gradio/atoms";
	import type { FileData, Client } from "@gradio/client";
	import { Video, Download } from "@gradio/icons";
	import { uploadToHuggingFace } from "@gradio/utils";
	import { DownloadLink } from "@gradio/wasm/svelte";

	import Player from "./Player.svelte";
	import type { I18nFormatter } from "@gradio/utils";

	export let value: FileData | null = null;
	export let subtitle: FileData | null = null;
	export let timestamps;
	export let marker_labels;
	export let start_times;
	export let end_times;
	export let descriptions;
	export let label: string | undefined = undefined;
	export let show_label = true;
	export let autoplay: boolean;
	export let show_share_button = true;
	export let show_download_button = true;
	export let loop: boolean;
	export let i18n: I18nFormatter;
	export let upload: Client["upload"];
	export let display_icon_button_wrapper_top_corner = false;

	let old_value: FileData | null = null;
	let old_subtitle: FileData | null = null;

	const dispatch = createEventDispatcher<{
		change: FileData;
		play: undefined;
		pause: undefined;
		end: undefined;
		stop: undefined;
		load: undefined;
	}>();

	$: value && dispatch("change", value);

	afterUpdate(async () => {
		// needed to bust subtitle caching issues on Chrome
		if (
			value !== old_value &&
			subtitle !== old_subtitle &&
			old_subtitle !== null
		) {
			old_value = value;
			value = null;
			await tick();
			value = old_value;
		}
		old_value = value;
		old_subtitle = subtitle;
	});
</script>

<BlockLabel {show_label} Icon={Video} label={label || "Video"} />
{#if !value || value.url === undefined}
	<Empty unpadded_box={true} size="large"><Video /></Empty>
{:else}
	{#key value.url}
		<Player
			src={value.url}
			subtitle={subtitle?.url}
			is_stream={value.is_stream}
			timestamps={timestamps}
			marker_labels={marker_labels}
			start_times={start_times}
			end_times={end_times}
			descriptions={descriptions}
			{autoplay}
			on:play
			on:pause
			on:stop
			on:end
			on:loadedmetadata={() => {
				// Deal with `<video>`'s `loadedmetadata` event as `VideoPreview`'s `load` event
				// to represent not only the video is loaded but also the metadata is loaded
				// so its dimensions (w/h) are known. This is used for Chatbot's auto scroll.
				dispatch("load");
			}}
			mirror={false}
			{label}
			{loop}
			interactive={false}
			{upload}
			{i18n}
		/>
	{/key}
	<div class="icon-buttons" data-testid="download-div">
		<IconButtonWrapper
			display_top_corner={display_icon_button_wrapper_top_corner}
		>
			{#if show_download_button}
				<DownloadLink
					href={value.is_stream
						? value.url?.replace("playlist.m3u8", "playlist-file")
						: value.url}
					download={value.orig_name || value.path}
				>
					<IconButton Icon={Download} label="Download" />
				</DownloadLink>
			{/if}
			{#if show_share_button}
				<ShareButton
					{i18n}
					on:error
					on:share
					{value}
					formatter={async (value) => {
						if (!value) return "";
						let url = await uploadToHuggingFace(value.data, "url");
						return url;
					}}
				/>
			{/if}
		</IconButtonWrapper>
	</div>
{/if}

<style>
	.icon-buttons {
		display: flex;
		position: absolute;
		top: 6px;
		right: 6px;
		gap: var(--size-1);
	}
</style>
