<script>
  import {
    Divider,
    Icon,
    Menu,
    MenuButton,
    MenuItem,
    SelectButton,
    SelectButtonGroup,
    Spacer,
    Toolbar,
  } from '@sveltia/ui';
  import equal from 'fast-deep-equal';
  import { _ } from 'svelte-i18n';
  import { writable } from 'svelte/store';
  import { getLocaleLabel } from '$lib/services/i18n';
  import { entryDraft, entryEditorSettings, revertChanges } from '$lib/services/contents/editor';
  import CopyMenuItems from '$lib/components/contents/details/editor/copy-menu-items.svelte';

  /**
   * @type {import('svelte/store').Writable<?EntryEditorPane>}
   */
  export let thisPane = writable(null);
  /**
   * @type {import('svelte/store').Writable<?EntryEditorPane>}
   */
  export let thatPane = writable(null);

  $: ({ collection, collectionFile, currentValues, originalValues, validities } =
    $entryDraft ?? /** @type {EntryDraft} */ ({}));
  $: ({ hasLocales = false, locales = ['default'] } =
    collection._i18n ?? /** @type {I18nConfig} */ ({}));
  $: otherLocales = hasLocales ? locales.filter((l) => l !== $thisPane?.locale) : [];
  $: canPreview =
    collection?.editor?.preview !== false && collectionFile?.editor?.preview !== false;
  $: canCopy = !!otherLocales.length;
  $: canRevert = !equal(currentValues[$thisPane?.locale], originalValues[$thisPane?.locale]);
</script>

<div class="header">
  <Toolbar class="secondary">
    {#if hasLocales}
      <!-- @todo Use a dropdown list when there are 5+ locales. -->
      <SelectButtonGroup>
        {#each locales as locale}
          {@const localeLabel = getLocaleLabel(locale)}
          {@const invalid = Object.values(validities[locale]).some(({ valid }) => !valid)}
          {#if !($thatPane?.mode === 'edit' && $thatPane?.locale === locale)}
            <SelectButton
              selected={$thisPane?.mode === 'edit' && $thisPane?.locale === locale}
              class="tertiary small {invalid ? 'error' : ''}"
              label={localeLabel}
              on:click={() => {
                $thisPane = { mode: 'edit', locale };

                if ($thatPane?.mode === 'preview') {
                  $thatPane = { mode: 'preview', locale };
                }
              }}
            >
              {#if invalid}
                <Icon slot="end-icon" name="error" label={$_('error')} />
              {/if}
            </SelectButton>
          {/if}
        {/each}
        {#if $thatPane?.mode === 'edit' && canPreview && $entryEditorSettings.showPreview}
          <SelectButton
            selected={$thisPane?.mode === 'preview'}
            class="tertiary small"
            label={$_('preview')}
            on:click={() => {
              $thisPane = { mode: 'preview', locale: $thatPane?.locale };
            }}
          />
        {/if}
      </SelectButtonGroup>
    {:else}
      <h3>{$thisPane?.mode === 'preview' ? $_('preview') : $_('edit')}</h3>
    {/if}
    <Spacer flex={true} />
    {#if $thisPane?.mode === 'edit'}
      <MenuButton class="ghost iconic" popupPosition="bottom-right">
        <Icon slot="start-icon" name="more_vert" label={$_('show_menu')} />
        <Menu slot="popup">
          {#if canCopy}
            <CopyMenuItems locale={$thisPane?.locale} translate={true} />
            {#if otherLocales.length > 1}
              <Divider />
            {/if}
            <CopyMenuItems locale={$thisPane?.locale} />
            <Divider />
          {/if}
          <MenuItem
            label={$_('revert_changes')}
            disabled={!canRevert}
            on:click={() => {
              revertChanges($thisPane?.locale);
            }}
          />
        </Menu>
      </MenuButton>
    {/if}
  </Toolbar>
</div>

<style lang="scss">
  .header {
    flex: none !important;
    background-color: var(--sui-tertiary-background-color);

    & > :global([role='toolbar']) {
      margin-right: auto;
      margin-left: auto;
      max-width: 800px;

      :global(h3) {
        font-size: var(--sui-font-size-default);
      }

      :global(button.error) {
        color: var(--sui-error-foreground-color);
      }
    }
  }
</style>