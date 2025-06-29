<script lang="ts">
    import type { TabsObj } from "../../../types/Tabs"
    import { activeEdit, activeShow, media, outputs } from "../../stores"
    import { clone } from "../helpers/array"
    import Icon from "../helpers/Icon.svelte"
    import { getExtension, getMediaType } from "../helpers/media"
    import { getActiveOutputs, setOutput } from "../helpers/output"
    import { getFilters } from "../helpers/style"
    import T from "../helpers/T.svelte"
    import { removeStore, updateStore } from "../helpers/update"
    import Button from "../inputs/Button.svelte"
    import Tabs from "../main/Tabs.svelte"
    import { addFilterString } from "./scripts/textStyle"
    import EditValues from "./tools/EditValues.svelte"
    import { croppingEdit, mediaEdits, mediaFilters, videoEdit } from "./values/media"

    let tabs: TabsObj = {
        media: { name: "items.media", icon: "image" },
        filters: { name: "edit.filters", icon: "filter" }
        // options: { name: "", icon: "options" },
    }
    let active: string = Object.keys(tabs)[0]

    // update values
    $: mediaId = $activeEdit.id || $activeShow!.id
    $: currentMedia = $media[mediaId] || {}

    let edits = clone(mediaEdits.media?.edit)!
    let filterEdits = clone(mediaFilters.media?.edit)!

    $: mediaType = $activeEdit.type === "camera" ? "camera" : getMediaType(getExtension(mediaId))

    // WIP camera / video cropping ??

    $: isVideo = mediaType === "video"
    $: isImage = mediaType === "image"
    $: isCamera = mediaType === "camera"
    $: if (isVideo) addVideoOptions()
    else if (isImage) addImageOptions()
    else if (isCamera) setCameraOptions()
    else edits = clone(mediaEdits.media?.edit)!
    function addVideoOptions() {
        if (!edits) return
        edits.video = clone(videoEdit)
    }
    function addImageOptions() {
        if (!edits) return
        edits.cropping = clone(croppingEdit)
    }
    function setCameraOptions() {
        edits.default[0].hidden = true
        const blurIndex = edits.default[1].values.options.findIndex((a) => a.id === "blur")
        if (blurIndex > -1) edits.default[1].values.options.splice(blurIndex, 1)
    }

    $: if (mediaId && isVideo) getVideoDuration()
    function getVideoDuration() {
        let video = document.createElement("video")
        video.setAttribute("src", mediaId)
        video.addEventListener("loadedmetadata", loaded)

        function loaded() {
            let videoDuration = video?.duration || 0
            if (!videoDuration) return

            edits.video[3].value = currentMedia?.toTime || videoDuration
            edits.video[2].values = { max: videoDuration }
            edits.video[3].values = { max: videoDuration }
        }
    }

    // set values
    $: if (currentMedia) {
        edits.default[0].value = currentMedia?.videoType || ""
        edits.default[1].value = currentMedia.fit || ""
        edits.default[2].value = currentMedia.flipped || false
        edits.default[3].value = currentMedia.flippedY || false
        if (edits.video) {
            edits.video[0].value = currentMedia.speed || "1"
            edits.video[1].value = currentMedia.volume ?? 100
            edits.video[2].value = currentMedia.fromTime || 0
            edits.video[3].value = currentMedia.toTime || edits.video[3].value
        }
        if (edits.cropping) {
            edits.cropping[0].value = currentMedia.cropping?.top || 0
            edits.cropping[1].value = currentMedia.cropping?.right || 0
            edits.cropping[2].value = currentMedia.cropping?.bottom || 0
            edits.cropping[3].value = currentMedia.cropping?.left || 0
        }

        // update filters
        let filters = getFilters(currentMedia.filter || "")
        let defaultFilters = mediaFilters.media?.edit?.default || []
        filterEdits.default.forEach((filter) => {
            let value = filters[filter.key || ""] ?? defaultFilters.find((a) => a.key === filter.key)?.value
            let index = filterEdits.default.findIndex((a) => a.key === filter.key)
            filterEdits.default[index].value = value
        })
    }

    function reset() {
        let deleteKeys: string[] = ["flipped", "flippedY", "fit", "speed", "volume", "fromTime", "toTime", "videoType", "cropping"]

        // reset
        if (active === "filters") deleteKeys = ["filter"]
        else if (active !== "media") return
        deleteKeys.forEach((key) => removeStore("media", { keys: [mediaId, key] }))

        // update output
        let currentOutput: any = $outputs[getActiveOutputs()[0]]
        let bg = currentOutput?.out?.background
        if (!bg) return
        deleteKeys.forEach((key) => delete bg[key])
        setOutput("background", bg)
    }

    export function valueChanged(input: any) {
        if (!mediaId) return

        let value = input.value
        if (value?.id !== undefined) value = value.id
        if (input.id === "filter") value = addFilterString(currentMedia?.filter || "", [input.key, value])

        updateStore("media", { keys: [mediaId, ...input.id.split(".")], value })

        // update output filters
        let currentOutput = $outputs[getActiveOutputs()[0]] || {}
        if (!currentOutput.out?.background || currentOutput.out?.background?.path !== mediaId) return

        let bg = currentOutput.out.background
        bg[input.id] = value
        setOutput("background", bg)
    }
</script>

<div class="main border editTools">
    <Tabs {tabs} bind:active />
    <div class="content">
        {#if active === "media"}
            <EditValues {edits} on:change={(e) => valueChanged(e.detail)} />
        {:else if active === "filters"}
            <EditValues edits={filterEdits} on:change={(e) => valueChanged(e.detail)} />
        {/if}
    </div>

    <span style="display: flex;">
        <Button style="flex: 1;" on:click={reset} dark center>
            <Icon id="reset" right />
            <T id={"actions.reset"} />
        </Button>
    </span>
</div>

<style>
    .main {
        display: flex;
        flex-direction: column;
        overflow: hidden;
        height: 100%;
    }

    .content {
        height: 100%;
        overflow-y: auto;
        overflow-x: hidden;
    }
</style>
