<!--
Copyright: Ankitects Pty Ltd and contributors
License: GNU AGPL, version 3 or later; http://www.gnu.org/licenses/agpl.html
-->
<script context="module" lang="ts">
    import type { ContentEditableAPI } from "../../editable/ContentEditable.svelte";
    import type { InputHandlerAPI } from "../../sveltelib/input-handler";
    import type { EditingInputAPI, FocusableInputAPI } from "../EditingArea.svelte";
    import type { SurroundedAPI } from "../surround";
    import type CustomStyles from "./CustomStyles.svelte";

    export interface RichTextInputAPI extends EditingInputAPI, SurroundedAPI {
        name: "rich-text";
        /** This is the contentEditable anki-editable element */
        element: Promise<HTMLElement>;
        moveCaretToEnd(): void;
        toggle(): boolean;
        preventResubscription(): () => void;
        inputHandler: InputHandlerAPI;
        /** The API exposed by the editable component */
        editable: ContentEditableAPI;
        customStyles: Promise<CustomStyles>;
    }

    function editingInputIsRichText(
        editingInput: EditingInputAPI | null,
    ): editingInput is RichTextInputAPI {
        return editingInput?.name === "rich-text";
    }

    import { registerPackage } from "../../lib/runtime-require";
    import contextProperty from "../../sveltelib/context-property";
    import lifecycleHooks from "../../sveltelib/lifecycle-hooks";
    import { Surrounder } from "../surround";

    const key = Symbol("richText");
    const [context, setContextProperty] = contextProperty<RichTextInputAPI>(key);
    const [globalInputHandler, setupGlobalInputHandler] = useInputHandler();
    const [lifecycle, instances, setupLifecycleHooks] =
        lifecycleHooks<RichTextInputAPI>();
    const surrounder = Surrounder.make();

    registerPackage("anki/RichTextInput", {
        context,
        surrounder,
        lifecycle,
        instances,
    });

    export {
        context,
        editingInputIsRichText,
        globalInputHandler as inputHandler,
        surrounder,
    };
</script>

<script lang="ts">
    import { getAllContexts, getContext, onMount } from "svelte";
    import type { Readable } from "svelte/store";

    import { placeCaretAfterContent } from "../../domlib/place-caret";
    import ContentEditable from "../../editable/ContentEditable.svelte";
    import { directionKey, fontFamilyKey, fontSizeKey } from "../../lib/context-keys";
    import { promiseWithResolver } from "../../lib/promise";
    import { singleCallback } from "../../lib/typing";
    import useDOMMirror from "../../sveltelib/dom-mirror";
    import useInputHandler from "../../sveltelib/input-handler";
    import { pageTheme } from "../../sveltelib/theme";
    import { context as editingAreaContext } from "../EditingArea.svelte";
    import { context as noteEditorContext } from "../NoteEditor.svelte";
    import getNormalizingNodeStore from "./normalizing-node-store";
    import useRichTextResolve from "./rich-text-resolve";
    import RichTextStyles from "./RichTextStyles.svelte";
    import { fragmentToStored, storedToFragment } from "./transform";

    export let hidden: boolean;

    const { focusedInput } = noteEditorContext.get();
    const { content, editingInputs } = editingAreaContext.get();

    const fontFamily = getContext<Readable<string>>(fontFamilyKey);
    const fontSize = getContext<Readable<number>>(fontSizeKey);
    const direction = getContext<Readable<"ltr" | "rtl">>(directionKey);

    const nodes = getNormalizingNodeStore();
    const [richTextPromise, resolve] = useRichTextResolve();
    const { mirror, preventResubscription } = useDOMMirror();
    const [inputHandler, setupInputHandler] = useInputHandler();
    const [customStyles, stylesResolve] = promiseWithResolver<CustomStyles>();

    export function attachShadow(element: Element): void {
        element.attachShadow({ mode: "open" });
    }

    async function moveCaretToEnd(): Promise<void> {
        const richText = await richTextPromise;
        if (richText.textContent?.length === 0) {
            // Calling this method when richText is empty will cause the first keystroke of
            // ibus-based input methods with candidates to go double. For example, if you
            // type "a" it becomes "aa". This problem exists in many linux distributions.
            // When richText is empty, there is no need to place the caret, just return.
            return;
        }

        placeCaretAfterContent(richText);
    }

    async function focus(): Promise<void> {
        const richText = await richTextPromise;
        richText.focus();
    }

    async function refocus(): Promise<void> {
        const richText = await richTextPromise;
        richText.blur();
        richText.focus();
        moveCaretToEnd();
    }

    function toggle(): boolean {
        hidden = !hidden;
        return hidden;
    }

    const className = "rich-text-editable";
    let richTextDiv: HTMLElement;

    async function getInputAPI(target: EventTarget): Promise<FocusableInputAPI | null> {
        if (target === richTextDiv) {
            return api;
        }

        return null;
    }

    export const api: RichTextInputAPI = {
        name: "rich-text",
        element: richTextPromise,
        focus,
        refocus,
        focusable: !hidden,
        toggle,
        getInputAPI,
        moveCaretToEnd,
        preventResubscription,
        inputHandler,
        editable: {} as ContentEditableAPI,
        customStyles,
    };

    const allContexts = getAllContexts();

    function attachContentEditable(element: Element, { stylesDidLoad }): void {
        (async () => {
            await stylesDidLoad;

            new ContentEditable({
                target: element.shadowRoot!,
                props: {
                    nodes,
                    resolve,
                    mirrors: [mirror],
                    inputHandlers: [setupInputHandler, setupGlobalInputHandler],
                    api: api.editable,
                },
                context: allContexts,
            });
        })();
    }

    function pushUpdate(isFocusable: boolean): void {
        api.focusable = isFocusable;
        $editingInputs = $editingInputs;
    }

    function setFocus(): void {
        $focusedInput = api;
        surrounder.enable(api);

        // We do not unset focusedInput here.
        // If we did, UI components for the input would react the store
        // being unset, even though most likely it will be set to some other
        // field right away.
    }

    function removeFocus(): void {
        surrounder.disable();
    }

    $: pushUpdate(!hidden);

    onMount(() => {
        $editingInputs.push(api);
        $editingInputs = $editingInputs;

        return singleCallback(
            content.subscribe((html: string): void =>
                nodes.setUnprocessed(storedToFragment(html)),
            ),
            nodes.subscribe((fragment: DocumentFragment): void =>
                content.set(fragmentToStored(fragment)),
            ),
        );
    });

    setContextProperty(api);
    setupLifecycleHooks(api);
</script>

<div class="rich-text-input" {hidden} on:focusin={setFocus} on:focusout={removeFocus}>
    <RichTextStyles
        color={$pageTheme.isDark ? "white" : "black"}
        fontFamily={$fontFamily}
        fontSize={$fontSize}
        direction={$direction}
        callback={stylesResolve}
        let:attachToShadow={attachStyles}
        let:stylesDidLoad
    >
        <div
            bind:this={richTextDiv}
            class={className}
            class:night-mode={$pageTheme.isDark}
            use:attachShadow
            use:attachStyles
            use:attachContentEditable={{ stylesDidLoad }}
            on:focusin
            on:focusout
        />

        {#await Promise.all([richTextPromise, stylesDidLoad]) then _}
            <div class="rich-text-widgets">
                <slot />
            </div>
        {/await}
    </RichTextStyles>
</div>

<style lang="scss">
    .rich-text-input {
        position: relative;
        padding: 6px;
    }

    .hidden {
        display: none;
    }
</style>
