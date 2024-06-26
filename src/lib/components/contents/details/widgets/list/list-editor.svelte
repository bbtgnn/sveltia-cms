<!--
  @component
  Implement the editor for the List widget.
  @see https://decapcms.org/docs/widgets/#list
-->
<script>
  import { Button, Group, Icon, Spacer, TextInput } from '@sveltia/ui';
  import { unflatten } from 'flat';
  import { onMount } from 'svelte';
  import { _ } from 'svelte-i18n';
  import FieldEditor from '$lib/components/contents/details/editor/field-editor.svelte';
  import AddItemButton from '$lib/components/contents/details/widgets/object/add-item-button.svelte';
  import ObjectHeader from '$lib/components/contents/details/widgets/object/object-header.svelte';
  import { entryDraft, getDefaultValues, updateListField } from '$lib/services/contents/editor';
  import { getFieldDisplayValue } from '$lib/services/contents/entry';
  import { defaultI18nConfig, getCanonicalLocale } from '$lib/services/contents/i18n';
  import { escapeRegExp, generateUUID } from '$lib/services/utils/strings';

  /**
   * @type {LocaleCode}
   */
  export let locale;
  /**
   * @type {string}
   */
  export let keyPath;
  /**
   * @type {string}
   */
  export let fieldId;
  /**
   * @type {string}
   */
  // svelte-ignore unused-export-let
  export let fieldLabel;
  /**
   * @type {ListField}
   */
  export let fieldConfig;
  /**
   * @type {string[]}
   */
  export let currentValue;
  /**
   * @type {boolean}
   */
  export let readonly = false;
  /**
   * @type {boolean}
   */
  export let required = false;
  /**
   * @type {boolean}
   */
  export let invalid = false;

  $: ({
    name: fieldName,
    label,
    i18n,
    // Widget-specific options
    allow_add: allowAdd = true,
    collapsed = false,
    summary,
    minimize_collapsed: minimizeCollapsed = false,
    label_singular: labelSingular,
    field,
    fields,
    // max,
    // min,
    add_to_top: addToTop = false,
    types,
    typeKey = 'type',
  } = fieldConfig);
  $: hasSingleSubField = !!field;
  $: hasMultiSubFields = Array.isArray(fields);
  $: hasVariableTypes = Array.isArray(types);
  $: hasSubFields = hasSingleSubField || hasMultiSubFields || hasVariableTypes;
  $: keyPathRegex = new RegExp(`^${escapeRegExp(keyPath)}\\.(\\d+)(.*)?`);
  $: ({ collectionName, fileName, collection, collectionFile, currentValues } =
    $entryDraft ?? /** @type {EntryDraft} */ ({}));
  $: ({ defaultLocale } = (collectionFile ?? collection)?._i18n ?? defaultI18nConfig);
  $: isDuplicateField = locale !== defaultLocale && i18n === 'duplicate';
  $: valueMap = currentValues[locale];
  $: canonicalLocale = getCanonicalLocale(locale);
  $: listFormatter = new Intl.ListFormat(canonicalLocale, { style: 'narrow', type: 'conjunction' });

  /** @type {{ [key: string]: any }[]} */
  $: items =
    unflatten(
      Object.fromEntries(
        Object.entries(valueMap)
          .filter(([_keyPath]) => _keyPath.match(keyPathRegex))
          .map(([_keyPath, value]) => [
            _keyPath.replace(new RegExp(`^${escapeRegExp(keyPath)}`), fieldName),
            value,
          ]),
      ),
    )[fieldName] ?? [];

  $: parentExpanded = !minimizeCollapsed;

  let mounted = false;
  let widgetId = '';
  let inputValue = '';

  onMount(() => {
    mounted = true;
    widgetId = generateUUID('short');

    items.forEach((__, index) => {
      /** @type {EntryDraft} */ ($entryDraft).viewStates[locale][`${keyPath}.${index}.expanded`] =
        !collapsed;
    });
  });

  /**
   * Update the input field value when the {@link currentValue} is reverted. This also cleans up the
   * input field value by removing extra spaces or commas.
   */
  const updateInputValue = () => {
    const currentValueStr = currentValue.join(', ');

    if (!inputValue.match(/,\s*$/) && inputValue.trim() !== currentValueStr) {
      inputValue = currentValueStr;
    }
  };

  /**
   * Update the value for the List widget w/o subfield(s). This has to be called from the `input`
   * event handler on `<TextInput>`, not a `inputValue` reaction, because it causes an infinite loop
   * due to {@link updateInputValue}.
   */
  const updateSimpleList = () => {
    const normalizedValue = inputValue
      .split(/,/g)
      .map((val) => val.trim())
      .filter((val) => val !== '');

    Object.keys($entryDraft?.currentValues ?? {}).forEach((_locale) => {
      if (i18n !== 'duplicate' && _locale !== locale) {
        return;
      }

      Object.keys($entryDraft?.currentValues[_locale] ?? {}).forEach((_keyPath) => {
        if (_keyPath.match(`^${escapeRegExp(keyPath)}\\.\\d+$`)) {
          delete $entryDraft?.currentValues[_locale][_keyPath];
        }
      });

      normalizedValue.forEach((val, index) => {
        /** @type {EntryDraft} */ ($entryDraft).currentValues[_locale][`${keyPath}.${index}`] = val;
      });
    });
  };

  /**
   * Update the value for the List widget with subfield(s).
   * @param {(arg: { valueList: object[], viewList: object[] }) => void} manipulate - See
   * {@link updateListField}.
   */
  const updateComplexList = (manipulate) => {
    Object.keys($entryDraft?.currentValues ?? {}).forEach((_locale) => {
      if (!(i18n !== 'duplicate' && _locale !== locale)) {
        updateListField(_locale, keyPath, manipulate);
      }
    });
  };

  /**
   * Add a new subfield to the list.
   * @param {string} [typeName] - Variable type name. If the field doesn’t have variable types, it
   * will be `undefined`.
   * @see https://decapcms.org/docs/variable-type-widgets/
   */
  const addItem = (typeName) => {
    updateComplexList(({ valueList, viewList }) => {
      const subFields = typeName
        ? types?.find(({ name }) => name === typeName)?.fields ?? []
        : fields ?? (field ? [field] : []);

      const index = addToTop ? 0 : valueList.length;
      const newItem = unflatten(getDefaultValues(subFields));

      if (typeName) {
        newItem[typeKey] = typeName;
      }

      valueList.splice(index, 0, hasSingleSubField && field ? newItem[field.name] : newItem);
      viewList.splice(index, 0, { expanded: true });
    });
  };

  /**
   * Remove a subfield.
   * @param {number} index - Target index.
   */
  const removeItem = (index) => {
    updateComplexList(({ valueList, viewList }) => {
      valueList.splice(index, 1);
      viewList.splice(index, 1);
    });
  };

  /**
   * Swap a subfield with the previous one.
   * @param {number} index - Target index.
   */
  const moveUpItem = (index) => {
    updateComplexList(({ valueList, viewList }) => {
      [valueList[index], valueList[index - 1]] = [valueList[index - 1], valueList[index]];
      [viewList[index], viewList[index - 1]] = [viewList[index - 1], viewList[index]];
    });
  };

  /**
   * Swap a subfield with the next one.
   * @param {number} index - Target index.
   */
  const moveDownItem = (index) => {
    updateComplexList(({ valueList, viewList }) => {
      [valueList[index], valueList[index + 1]] = [valueList[index + 1], valueList[index]];
      [viewList[index], viewList[index + 1]] = [viewList[index + 1], viewList[index]];
    });
  };

  $: {
    if (mounted && !hasSubFields) {
      void currentValue;
      updateInputValue();
    }
  }

  /**
   * Format the summary template.
   * @param {{ [key: string]: any }} item - List item.
   * @param {number} index - List index.
   * @param {string} [summaryTemplate] - Summary template, e.g. `{{fields.slug}}`.
   * @returns {string} Formatted summary.
   */
  const formatSummary = (item, index, summaryTemplate) => {
    if (!summaryTemplate) {
      return item.title || item.name || '';
    }

    return summaryTemplate.replaceAll(/{{fields\.(.+?)}}/g, (_match, _fieldName) => {
      const value = getFieldDisplayValue({
        collectionName,
        fileName,
        valueMap,
        keyPath: hasSingleSubField ? `${keyPath}.${index}` : `${keyPath}.${index}.${_fieldName}`,
        locale,
      });

      return Array.isArray(value) ? listFormatter.format(value) : String(value);
    });
  };
</script>

<Group aria-labelledby="list-{widgetId}-summary">
  {#if hasSubFields}
    <div role="none" class="toolbar top">
      <Button
        iconic
        disabled={!items.length}
        aria-label={parentExpanded ? $_('collapse') : $_('expand')}
        aria-expanded={parentExpanded}
        aria-controls="list-{widgetId}-item-list"
        on:click={() => {
          parentExpanded = !parentExpanded;
        }}
      >
        <Icon slot="start-icon" name={parentExpanded ? 'expand_more' : 'chevron_right'} />
      </Button>
      <div role="none" class="summary" id="object-{widgetId}-summary">
        {items.length}
        {(items.length === 1 ? labelSingular : undefined) || label || fieldName}
      </div>
      <Spacer flex />
      {#if allowAdd && (addToTop || !items.length)}
        <AddItemButton disabled={isDuplicateField} {fieldConfig} {items} {addItem} />
      {/if}
    </div>
    <div
      role="none"
      id="list-{widgetId}-item-list"
      class="item-list"
      class:collapsed={!parentExpanded}
    >
      {#each items as item, index}
        {@const expanded = !!$entryDraft?.viewStates[locale][`${keyPath}.${index}.expanded`]}
        {@const typeConfig = hasVariableTypes
          ? types?.find(({ name }) => name === item[typeKey])
          : undefined}
        {@const subFields = hasVariableTypes
          ? typeConfig?.fields ?? []
          : fields ?? (field ? [field] : [])}
        {@const summaryTemplate = hasVariableTypes ? typeConfig?.summary || summary : summary}
        <!-- @todo Support drag sorting. -->
        <div role="none" class="item">
          <ObjectHeader
            label={hasVariableTypes ? typeConfig?.label || typeConfig?.name : ''}
            controlId="list-{widgetId}-item-{index}-body"
            {expanded}
            toggleExpanded={() => {
              Object.keys($entryDraft?.viewStates ?? {}).forEach((_locale) => {
                /** @type {EntryDraft} */ ($entryDraft).viewStates[_locale][
                  `${keyPath}.${index}.expanded`
                ] = !expanded;
              });
            }}
            removeButtonVisible={true}
            removeButtonDisabled={isDuplicateField}
            remove={() => {
              removeItem(index);
            }}
          >
            <svelte:fragment slot="middle">
              <Button
                size="small"
                iconic
                disabled={isDuplicateField || index === 0}
                aria-label={$_('move_up')}
                on:click={() => {
                  moveUpItem(index);
                }}
              >
                <Icon slot="start-icon" name="arrow_upward" />
              </Button>
              <Spacer />
              <Button
                iconic
                size="small"
                disabled={isDuplicateField || index === items.length - 1}
                aria-label={$_('move_down')}
                on:click={() => {
                  moveDownItem(index);
                }}
              >
                <Icon slot="start-icon" name="arrow_downward" />
              </Button>
            </svelte:fragment>
          </ObjectHeader>
          <div role="none" class="item-body" id="list-{widgetId}-item-{index}-body">
            {#if expanded}
              {#each subFields as subField (subField.name)}
                <FieldEditor
                  keyPath={hasSingleSubField
                    ? `${keyPath}.${index}`
                    : `${keyPath}.${index}.${subField.name}`}
                  {locale}
                  fieldConfig={subField}
                />
              {/each}
            {:else}
              <div role="none" class="summary">
                {formatSummary(item, index, summaryTemplate)}
              </div>
            {/if}
          </div>
        </div>
      {/each}
    </div>
    {#if allowAdd && !addToTop && items.length}
      <div role="none" class="toolbar bottom">
        <AddItemButton disabled={isDuplicateField} {fieldConfig} {items} {addItem} />
        <Spacer flex />
      </div>
    {/if}
  {:else}
    <TextInput
      bind:value={inputValue}
      flex
      {readonly}
      {required}
      {invalid}
      aria-errormessage="{fieldId}-error"
      on:input={() => {
        updateSimpleList();
      }}
    />
  {/if}
</Group>

<style lang="scss">
  .toolbar {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .item-list {
    &.collapsed {
      display: none;
    }
  }

  .item {
    margin: 8px 0;
    border-width: 2px;
    border-color: var(--sui-secondary-border-color);
    border-radius: var(--sui-control-medium-border-radius);

    .summary {
      overflow: hidden;
      padding: 8px;
      white-space: nowrap;
      text-overflow: ellipsis;
    }
  }
</style>
