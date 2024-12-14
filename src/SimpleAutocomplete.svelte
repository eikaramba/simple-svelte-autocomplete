<script>
    import { flip } from "svelte/animate"
    import { fade } from "svelte/transition"
    import { untrack } from "svelte";
    
    let filteredListItems = $state([]);
    let listItems = $state([]);
    let highlightIndex = $state(-1);
    let opened = $state(false);
    let filteredTextLength = $state(0);
    let lastRequestId = $state(0);
    let lastResponseId = $state(0);
    let loading = $state(false);

    let {
        // Props for items/data
        /**
       * function to use to get all items (alternative to providing items)
       * @type {boolean|function}
       */
        text = $bindable(undefined),
        items = [],
        searchFunction = undefined,
        labelFieldName = undefined,
        keywordsFieldName = labelFieldName, 
        valueFieldName = undefined,
        multiple = false,
        selectedItem = $bindable(multiple ? [] : undefined),
        highlightedItem = $bindable(),
        value = $bindable(undefined),

        // Functions for data handling 
        labelFunction = (item) => {
            if (item === undefined || item === null) return ""
            return labelFieldName ? item[labelFieldName] : item
        },

        keywordsFunction = (item) => {
            if (item === undefined || item === null) return ""
            return keywordsFieldName ? item[keywordsFieldName] : labelFunction(item)  
        },

        valueFunction = (item, forceSingle = false) => {
            if (item === undefined || item === null) return item
            if (!multiple || forceSingle) {
                return valueFieldName ? item[valueFieldName] : item
            } else {
                return item.map((i) => (valueFieldName ? i[valueFieldName] : i))
            }
        },

        keywordsCleanFunction = (keywords) => keywords,
        textCleanFunction = (userEnteredText) => userEnteredText,

        // Event handlers
        beforeChange = (oldSelectedItem, newSelectedItem) => true,
        onChange = (newSelectedItem) => {},
        onFocus = () => {},
        onBlur = () => {},
        onCreate = (text) => {
            if (debug) console.log("onCreate: " + text)
        },

        // Behavior props
        selectFirstIfEmpty = false,
        minCharactersToSearch = 1,
        maxItemsToShowInList = 0,
        create = false,
        ignoreAccents = true,
        matchAllKeywords = true,
        sortByMatchedKeywords = false,
        itemFilterFunction = undefined,
        itemSortFunction = undefined,
        lock = false,
        delay = 0,
        localFiltering = true,
        localSorting = true,
        cleanUserText = true,
        lowercaseKeywords = true,
        closeOnBlur = false,
        orderableSelection = false,

        // UI props
        hideArrow = false,
        showClear = false,
        clearText = "&#10006;",
        showLoadingIndicator = false,
        noResultsText = "No results found",
        loadingText = "Loading results...", 
        moreItemsText = "items not shown",
        createText = "Not found, add anyway?",
        placeholder = undefined,
        className = undefined,
        inputClassName = undefined,
        inputId = undefined,
        name = undefined, 
        selectName = undefined,
        selectId = undefined,
        title = undefined,
        html5autocomplete = undefined,
        autocompleteOffValue = "off",
        readonly = undefined,
        dropdownClassName = undefined,
        disabled = false,
        noInputStyles = false,
        required = null,
        debug = false,
        tabindex = 0,

        // Snippets
        noResults = undefined,
        item = undefined,
        dropdownFooter = undefined,
        loadingSlot = undefined,
        createSlot = undefined,
        dropdownHeader = undefined,


        ...rest
    } = $props();


    let input;
    let list;
    let inputContainer;

    // Track request/response state
    let inputDelayTimeout = $state();
    let setPositionOnNextUpdate = $state(false);

    const uniqueId = "sautocomplete-" + Math.floor(Math.random() * 1000);

    // Convert position update to effect
    $effect(() => {
        if(setPositionOnNextUpdate) {
            setScrollAwareListPosition()
        }
        setPositionOnNextUpdate = false;
    });
  
    // --- Functions ---
  
    function safeFunction(theFunction, argument) {
      if (typeof theFunction !== "function") {
        console.error("Not a function: " + theFunction + ", argument: " + argument)
        return undefined
      }
      let result
      try {
        result = theFunction(argument)
      } catch (error) {
        console.warn(
          "Error executing Autocomplete function on value: " + argument + " function: " + theFunction
        )
      }
      return result
    }
  
    function safeStringFunction(theFunction, argument) {
      let result = safeFunction(theFunction, argument)
      if (result === undefined || result === null) {
        result = ""
      }
      if (typeof result !== "string") {
        result = result.toString()
      }
      return result
    }
  
    function safeLabelFunction(item) {
      // console.log("labelFunction: " + labelFunction);
      // console.log("safeLabelFunction, item: " + item);
      return safeStringFunction(labelFunction, item)
    }
  
    function safeKeywordsFunction(item) {
      // console.log("safeKeywordsFunction");
      const keywords = safeStringFunction(keywordsFunction, item)
      let result = safeStringFunction(keywordsCleanFunction, keywords)
      result = lowercaseKeywords ? result.toLowerCase().trim() : result
      if (ignoreAccents) {
        result = removeAccents(result)
      }
  
      if (debug) {
        console.log("Extracted keywords: '" + result + "' from item: " + JSON.stringify(item))
      }
      return result
    }
  
    function prepareListItems() {
      let timerId
      if (debug) {
        timerId = `Autocomplete prepare list ${inputId ? `(id: ${inputId})` : ""}`
        console.time(timerId)
        console.log("Prepare items to search")
        console.log("items: " + JSON.stringify(items))
      }
  
      if (!Array.isArray(items)) {
        console.warn("Autocomplete items / search function did not return array but", items)
        items = []
      }
  
      const length = items ? items.length : 0
      listItems = new Array(length)
  
      if (length > 0) {
        items.forEach((item, i) => {
          const listItem = getListItem(item)
          if (listItem === undefined) {
            console.log("Undefined item for: ", item)
          }
          listItems[i] = listItem
        })
      }
  
      filteredListItems = listItems
  
      if (debug) {
        console.log(listItems.length + " items to search")
        console.timeEnd(timerId)
      }
    }
  
    function getListItem(item) {
      return {
        // keywords representation of the item
        keywords: localFiltering ? safeKeywordsFunction(item) : [],
        // item label
        label: safeLabelFunction(item),
        // store reference to the origial item
        item: item,
      }
    }
  
    // -- Reactivity --
    $effect(() => {
        if (!searchFunction && items) {
          untrack(() => prepareListItems())
        }
    });
    // Watch selectedItem changes
    $effect(() => {
            if(selectedItem)
            untrack(() => onSelectedItemChanged())
    });

    
    $effect(() => {
      highlightedItem = filteredListItems &&
                        highlightIndex &&
                        highlightIndex >= 0 &&
                        highlightIndex < filteredListItems.length
                          ? filteredListItems[highlightIndex].item
                          : null
    });


    function onSelectedItemChanged() {
      value = valueFunction(selectedItem)
      if (!multiple) {
        text = safeLabelFunction(selectedItem)
      }
  
      filteredListItems = listItems
      onChange(selectedItem)
    }
  
  
  
    let showList = $derived(opened && ((items?.length > 0) || filteredTextLength > 0));
  
    let hasSelection = $derived(
        (multiple && selectedItem?.length > 0) || (!multiple && selectedItem)
    );

    let clearable = $derived(showClear || ((lock || multiple) && hasSelection));

    let locked = $derived(lock && hasSelection);
  
    function prepareUserEnteredText(userEnteredText) {
      if (userEnteredText === undefined || userEnteredText === null) {
        return ""
      }
  
      if (!cleanUserText) {
        return userEnteredText
      }
  
      const textFiltered = userEnteredText.replace(/[&/\\#,+()$~%.'":*?<>{}]/g, " ").trim()
  
      const cleanUserEnteredText = safeStringFunction(textCleanFunction, textFiltered)
      const textTrimmed = lowercaseKeywords
        ? cleanUserEnteredText.toLowerCase().trim()
        : cleanUserEnteredText.trim()
  
      return textTrimmed
    }
  
    function numberOfMatches(listItem, searchWords) {
      if (!listItem) {
        return 0
      }
  
      const itemKeywords = listItem.keywords
  
      let matches = 0
      searchWords.forEach((searchWord) => {
        if (itemKeywords.includes(searchWord)) {
          matches++
        }
      })
  
      return matches
    }
  
    async function search() {
        let timerId;
        if (debug) {
            timerId = `Autocomplete search ${inputId ? `(id: ${inputId})` : ""}`;
            console.time(timerId);
            console.log("Searching user entered text: '" + text + "'");
        }

        let textFiltered = prepareUserEnteredText(text);
        if (minCharactersToSearch > 1 && textFiltered.length < minCharactersToSearch) {
            textFiltered = "";
        }
        filteredTextLength = textFiltered.length;

        if (debug) {
            console.log("Changed user entered text '" + text + "' into '" + textFiltered + "'");
        }

        // if no search text load all items
        if (textFiltered === "") {
            if (searchFunction) {
                items = [];
                if (debug) {
                    console.log("User entered text is empty clear list of items");
                }
            } else {
                filteredListItems = listItems;
                if (debug) {
                    console.log("User entered text is empty set the list of items to all items");
                }
            }
            if (closeIfMinCharsToSearchReached()) {
                if (debug) {
                    console.timeEnd(timerId);
                }
                return;
            }
        }

        if (!searchFunction) {
            // internal search
            processListItems(textFiltered);
        } else {
            // external search which provides items
            lastRequestId++;
            const currentRequestId = lastRequestId;
            loading = true;

            try {
                // searchFunction is a generator
                if (searchFunction.constructor.name === "AsyncGeneratorFunction") {
                    for await (const chunk of searchFunction(textFiltered, maxItemsToShowInList)) {
                        // a chunk of an old response: throw it away
                        if (currentRequestId < lastResponseId) {
                            return false;
                        }

                        // a chunk for a new response: reset the item list
                        if (currentRequestId > lastResponseId) {
                            items = [];
                        }

                        lastResponseId = currentRequestId;
                        items = [...items, ...chunk];
                        processListItems(textFiltered);
                    }

                    // there was nothing in the chunk
                    if (lastResponseId < currentRequestId) {
                        lastResponseId = currentRequestId;
                        items = [];
                        processListItems(textFiltered);
                    }
                }
                // searchFunction is a regular function
                else {
                    const result = await searchFunction(textFiltered, maxItemsToShowInList);

                    if (currentRequestId < lastResponseId) {
                        return false;
                    }

                    lastResponseId = currentRequestId;
                    items = result;
                    processListItems(textFiltered);
                }
            } finally {
                loading = false;
            }
        }

        if (debug) {
            console.timeEnd(timerId);
            console.log("Search found " + filteredListItems.length + " items");
        }
    }
  
    function defaultItemFilterFunction(listItem, searchWords) {
      const matches = numberOfMatches(listItem, searchWords)
      if (matchAllKeywords) {
        return matches >= searchWords.length
      } else {
        return matches > 0
      }
    }
  
    function defaultItemSortFunction(obj1, obj2, searchWords) {
      return numberOfMatches(obj2, searchWords) - numberOfMatches(obj1, searchWords)
    }
  
    function processListItems(textFiltered="") {
      // cleans, filters, orders, and highlights the list items
      prepareListItems()
  
      const textFilteredWithoutAccents = ignoreAccents ? removeAccents(textFiltered) : textFiltered
      const searchWords = textFilteredWithoutAccents.split(/\s+/g).filter((word) => word !== "")
  
      // local search
      let tempfilteredListItems
      if (localFiltering) {
        if (itemFilterFunction) {
          tempfilteredListItems = listItems.filter((item) =>
            itemFilterFunction(item.item, searchWords)
          )
        } else {
          tempfilteredListItems = listItems.filter((item) =>
            defaultItemFilterFunction(item, searchWords)
          )
        }
  
        if (localSorting) {
          if (itemSortFunction) {
            tempfilteredListItems = tempfilteredListItems.sort((item1, item2) =>
              itemSortFunction(item1.item, item2.item, searchWords)
            )
          } else {
            if (sortByMatchedKeywords) {
              tempfilteredListItems = tempfilteredListItems.sort((item1, item2) =>
                defaultItemSortFunction(item1, item2, searchWords)
              )
            }
          }
        }
      } else {
        tempfilteredListItems = listItems
      }
  
      const hlfilter = highlightFilter(searchWords, "label")
      filteredListItems = tempfilteredListItems.map(hlfilter)
      closeIfMinCharsToSearchReached()
      return true
    }
  
    // $: text, search();
  
    function afterCreate(createdItem) {
      let listItem
      if (debug) {
        console.log("createdItem", createdItem)
      }
      if ("undefined" !== typeof createdItem) {
        prepareListItems()
        filteredListItems = listItems
        let index = findItemIndex(createdItem, filteredListItems)
  
        // if the items array was not updated, add the created item manually
        if (index <= 0) {
          items = [createdItem]
          prepareListItems()
          filteredListItems = listItems
          index = 0
        }
  
        if (index >= 0) {
          highlightIndex = index
          listItem = filteredListItems[highlightIndex]
        }
      }
      return listItem
    }
  
    function selectListItem(listItem) {
      if (debug) {
        console.log("selectListItem", listItem)
      }
      if ("undefined" === typeof listItem && create) {
        // allow undefined items if create is enabled
        const createdItem = onCreate(text)
        if ("undefined" !== typeof createdItem) {
          if (typeof createdItem.then === "function") {
            createdItem.then((newItem) => {
              if ("undefined" !== typeof newItem) {
                const newListItem = afterCreate(newItem)
                if ("undefined" !== typeof newListItem) {
                  selectListItem(newListItem)
                }
              }
            })
            return true
          } else {
            listItem = afterCreate(createdItem)
          }
        }
      }
  
      if ("undefined" === typeof listItem) {
        if (debug) {
          console.log(`listItem is undefined. Can not select.`)
        }
        return false
      }
  
      if (locked) {
        return true
      }
  
      const newSelectedItem = listItem.item
      if (beforeChange(selectedItem, newSelectedItem)) {
        // simple selection
        if (!multiple) {
          selectedItem = undefined // triggers change even if the the same item is selected
          selectedItem = newSelectedItem
        }
        // first selection of multiple ones
        else if (!selectedItem) {
          selectedItem = [newSelectedItem]
        }
        // selecting something already selected => unselect it
        else if (selectedItem.includes(newSelectedItem)) {
          selectedItem = selectedItem.filter((i) => i !== newSelectedItem)
        }
        // adds the element to the selection
        else {
          selectedItem = [...selectedItem, newSelectedItem]
        }
      }
      return true
    }
  
    function selectItem() {
      if (debug) {
        console.log("selectItem", highlightIndex)
      }
      const listItem = filteredListItems[highlightIndex]
      if (selectListItem(listItem)) {
        if (debug) {
          console.log("selectListItem true, closing")
        }
        close()
        if (multiple) {
          text = ""
          input.focus()
        }
      } else {
        if (debug) {
          console.log("selectListItem false, not closing")
        }
      }
    }
  
    function up() {
      if (debug) {
        console.log("up")
      }
  
      open()
      if (highlightIndex > 0) {
        highlightIndex--
      }
  
      highlight()
    }
  
    function down() {
      if (debug) {
        console.log("down")
      }
  
      open()
      if (highlightIndex < filteredListItems.length - 1) {
        highlightIndex++
      }
  
      highlight()
    }
  
    function highlight() {
      if (debug) {
        console.log("highlight")
      }
  
      const query = ".selected"
      if (debug) {
        console.log("Seaching DOM element: " + query + " in " + list)
      }
  
      /**
       * @param {Element} el
       */
      const el = list && list.querySelector(query)
      if (el) {
        if (typeof el.scrollIntoViewIfNeeded === "function") {
          if (debug) {
            console.log("Scrolling selected item into view")
          }
          el.scrollIntoViewIfNeeded()
        } else if (el.scrollIntoView === "function") {
          if (debug) {
            console.log("Scrolling selected item into view")
          }
          el.scrollIntoView()
        } else {
          if (debug) {
            console.warn(
              "Could not scroll selected item into view, scrollIntoViewIfNeeded not supported"
            )
          }
        }
      } else {
        if (debug) {
          console.warn("Selected item not found to scroll into view")
        }
      }
    }
  
    function onListItemClick(listItem) {
      if (debug) {
        console.log("onListItemClick")
      }
  
      if (selectListItem(listItem)) {
        close()
        if (multiple) {
          text = ""
          input.focus()
        }
      }
    }
  
    function onDocumentClick(e) {
      if (debug) {
        console.log("onDocumentClick")
      }
      if (e.composedPath().some((path) => path.classList && path.classList.contains(uniqueId))) {
        if (debug) {
          console.log("onDocumentClick inside")
        }
        // resetListToAllItemsAndOpen();
        highlight()
      } else {
        if (debug) {
          console.log("onDocumentClick outside")
        }
        close()
      }
    }
  
    function onKeyDown(e) {
      if (debug) {
        console.log("onKeyDown")
      }
  
      let key = e.key
      if (key === "Tab" && e.shiftKey) key = "ShiftTab"
      const fnmap = {
        Tab: opened ? close : null,
        ShiftTab: opened ? close : null,
        ArrowDown: down.bind(this),
        ArrowUp: up.bind(this),
        Escape: onEsc.bind(this),
        Backspace: multiple && hasSelection && !text ? onBackspace.bind(this) : null,
      }
      const fn = fnmap[key]
      if (typeof fn === "function") {
        fn(e)
      }
    }
  
    function onKeyPress(e) {
      if (debug) {
        console.log("onKeyPress")
      }
  
      if (e.key === "Enter") {
        onEnter(e)
      }
    }
  
    function onEnter(e) {
      if (opened) {
        e.preventDefault()
        selectItem()
      }
    }
  
    function onInput(e) {
      if (debug) {
        console.log("onInput")
      }
  
      text = e.target.value
      if (inputDelayTimeout) {
        clearTimeout(inputDelayTimeout)
      }
  
      if (delay) {
        inputDelayTimeout = setTimeout(processInput, delay)
      } else {
        processInput()
      }
    }
  
    function unselectItem(tag) {
      if (debug) {
        console.log("unselectItem", tag)
      }
      selectedItem = selectedItem.filter((i) => i !== tag)
      input.focus()
    }
  
    function processInput() {
      if (search()) {
        highlightIndex = 0
        open()
      }
    }
  
    function onInputClick() {
      if (debug) {
        console.log("onInputClick")
      }
      resetListToAllItemsAndOpen()
    }
  
    function onEsc(e) {
      if (debug) {
        console.log("onEsc")
      }
  
      //if (text) return clear();
      e.stopPropagation()
      if (opened) {
        input.focus()
        close()
      }
    }
  
    function onBackspace(e) {
      if (debug) {
        console.log("onBackspace")
      }
  
      unselectItem(selectedItem[selectedItem.length - 1])
    }
  
    function onFocusInternal() {
      if (debug) {
        console.log("onFocus")
      }
  
      onFocus()
  
      resetListToAllItemsAndOpen()
    }
  
    function onBlurInternal() {
      if (debug) {
        console.log("onBlur")
      }
  
      if (closeOnBlur) {
        close()
      }
  
      onBlur()
    }
  
    function resetListToAllItemsAndOpen() {
      if (debug) {
        console.log("resetListToAllItemsAndOpen")
      }
  
      if (searchFunction && !listItems.length) {
        search()
      } else if (!text) {
        processListItems()
      }
  
      open()
  
      // find selected item
      if (selectedItem) {
        if (debug) {
          console.log("Searching currently selected item: " + JSON.stringify(selectedItem))
        }
  
        const index = findItemIndex(selectedItem, filteredListItems)
        if (index >= 0) {
          highlightIndex = index
          highlight()
        }
      }
    }
  
    function findItemIndex(item, items) {
      if (debug) {
        console.log("Finding index for item", item)
      }
      let index = -1
      for (let i = 0; i < items.length; i++) {
        const listItem = items[i]
        if ("undefined" === typeof listItem) {
          if (debug) {
            console.log(`listItem ${i} is undefined. Skipping.`)
          }
          continue
        }
        if (debug) {
          console.log("Item " + i + ": " + JSON.stringify(listItem))
        }
        if (item === listItem.item) {
          index = i
          break
        }
      }
  
      if (debug) {
        if (index >= 0) {
          console.log("Found index for item: " + index)
        } else {
          console.warn("Not found index for item: " + item)
        }
      }
      return index
    }
  
    function open() {
      if (debug) {
        console.log("open")
      }
  
      // check if the search text has more than the min chars required
      if (locked || notEnoughSearchText()) {
        return
      }
  
      setPositionOnNextUpdate = true
  
      opened = true
    }
  
    function close() {
      if (debug) {
        console.log("close")
      }
      opened = false
      loading = false
  
      if (!text && selectFirstIfEmpty) {
        highlightIndex = 0
        selectItem()
      }
    }
  
    function notEnoughSearchText() {
      return (
        minCharactersToSearch > 0 &&
        filteredTextLength < minCharactersToSearch &&
        // When no searchFunction is defined, the menu should always open when the input is focused
        (searchFunction || filteredTextLength > 0)
      )
    }
  
    function closeIfMinCharsToSearchReached() {
      if (notEnoughSearchText()) {
        close()
        return true
      }
      return false
    }
  
    function clear() {
      if (debug) {
        console.log("clear")
      }
  
      text = ""
      selectedItem = multiple ? [] : undefined
  
      setTimeout(() => {
        input.focus()
      })
    }
  
    export function highlightFilter(keywords, field) {
      return (item) => {
        let label = item[field]
  
        const newItem = Object.assign({ highlighted: undefined }, item)
        newItem.highlighted = label
  
        const labelLowercase = label.toLowerCase()
        const labelLowercaseNoAc = ignoreAccents ? removeAccents(labelLowercase) : labelLowercase
  
        if (keywords && keywords.length) {
          const positions = []
  
          for (let i = 0; i < keywords.length; i++) {
            let keyword = keywords[i]
            if (ignoreAccents) {
              keyword = removeAccents(keyword)
            }
            const keywordLen = keyword.length
  
            let pos1 = 0
            do {
              pos1 = labelLowercaseNoAc.indexOf(keyword, pos1)
              if (pos1 >= 0) {
                let pos2 = pos1 + keywordLen
                positions.push([pos1, pos2])
                pos1 = pos2
              }
            } while (pos1 !== -1)
          }
  
          if (positions.length > 0) {
            const keywordPatterns = new Set()
            for (let i = 0; i < positions.length; i++) {
              const pair = positions[i]
              const pos1 = pair[0]
              const pos2 = pair[1]
  
              const keywordPattern = labelLowercase.substring(pos1, pos2)
              keywordPatterns.add(keywordPattern)
            }
            for (let keywordPattern of keywordPatterns) {
              // FIXME pst: workarond for wrong replacement <b> tags
              if (keywordPattern === "b") {
                continue
              }
              const reg = new RegExp("(" + keywordPattern + ")", "ig")
  
              const newHighlighted = newItem.highlighted.replace(reg, "<b>$1</b>")
              newItem.highlighted = newHighlighted
            }
          }
        }
  
        return newItem
      }
    }
  
    function removeAccents(str) {
      return str.normalize("NFD").replace(/[\u0300-\u036f]/g, "")
    }
  
    function isConfirmed(listItem) {
      if (!selectedItem) {
        return false
      }
      if (multiple) {
        return selectedItem.includes(listItem)
      } else {
        return listItem === selectedItem
      }
    }
  
    let draggingOver = false
  
    function dragstart(event, index) {
      if (orderableSelection) {
        event.dataTransfer.setData("source", index)
      }
    }
  
    function dragover(event, index) {
      if (orderableSelection) {
        event.preventDefault()
        draggingOver = index
      }
    }
  
    function dragleave(event, index) {
      if (orderableSelection) {
        draggingOver = false
      }
    }
  
    function drop(event, index) {
      if (orderableSelection) {
        event.preventDefault()
        draggingOver = false
        let from = parseInt(event.dataTransfer.getData("source"))
        let to = index
        if (from != to) {
          moveSelectedItem(from, to)
        }
      }
    }
  
    function moveSelectedItem(from, to) {
      let newSelection = [...selectedItem]
      if (from < to) {
        newSelection.splice(to + 1, 0, newSelection[from])
        newSelection.splice(from, 1)
      } else {
        newSelection.splice(to, 0, newSelection[from])
        newSelection.splice(from + 1, 1)
      }
      selectedItem = newSelection
    }
  
    function setScrollAwareListPosition() {
      const { height: viewPortHeight } = window.visualViewport
      const { bottom: inputButtom, height: inputHeight } = inputContainer.getBoundingClientRect()
      const { height: listHeight } = list.getBoundingClientRect()
  
      if (inputButtom + listHeight > viewPortHeight) {
        list.style.top = `-${inputHeight + listHeight}px`
      } else {
        list.style.top = "0px"
      }
    }
  </script>
  <div
    class="{className ? className : ''} autocomplete select is-fullwidth {uniqueId}"
    class:hide-arrow={hideArrow || !items.length}
    class:is-multiple={multiple}
    class:show-clear={clearable}
    class:is-loading={showLoadingIndicator && loading}
  >
    <select name={selectName} id={selectId} {multiple}>
      {#if !multiple && selectedItem}
          <option value={valueFunction(selectedItem, true)} selected>
              {safeLabelFunction(selectedItem)}
          </option>
      {:else if multiple && selectedItem?.length}
          {#each selectedItem as i}
              <option value={valueFunction(i, true)} selected>
                  {safeLabelFunction(i)}
              </option>
          {/each}
      {/if}
  </select>
    <div class="input-container" bind:this={inputContainer}>
      {#if multiple && hasSelection}
        {#each selectedItem as tagItem, i (valueFunction(tagItem, true))}
          <div
            draggable={true}
            animate:flip={{ duration: 200 }}
            transition:fade={{ duration: 200 }}
            on:dragstart={(event) => dragstart(event, i)}
            on:dragover={(event) => dragover(event, i)}
            on:dragleave={(event) => dragleave(event, i)}
            on:drop={(event) => drop(event, i)}
            class:is-active={draggingOver === i}
          >
            {@render tag(safeLabelFunction(tagItem), tagItem, () => unselectItem(tagItem))}
            {#if !tag}
              <div class="tags has-addons">
                  <span class="tag">{label}</span>
                  <span
                      class="tag is-delete"
                      on:click|preventDefault={unselectItem}
                      on:keypress|preventDefault={(e) => {e.key == "Enter" && unselectItem()}}
                  />
              </div>
            {/if}
          </div>
        {/each}
      {/if}
      <input
        type="text"
        class="{inputClassName ? inputClassName : ''} {noInputStyles
          ? ''
          : 'input autocomplete-input'}"
        id={inputId ? inputId : ""}
        autocomplete={html5autocomplete ? "on" : autocompleteOffValue}
        {placeholder}
        {name}
        {disabled}
        {required}
        {title}
        readonly={readonly || locked}
        {tabindex}
        bind:this={input}
        bind:value={text}
        on:input={onInput}
        on:focus={onFocusInternal}
        on:blur={onBlurInternal}
        on:keydown={onKeyDown}
        on:click={onInputClick}
        on:keypress={onKeyPress}
        on:dragover={(event) => dragover(event, selectedItem.length - 1)}
        on:drop={(event) => drop(event, selectedItem.length - 1)}
        {...rest}
      />
      {#if clearable}
        <span
          on:click={clear}
          on:keypress={(e) => {e.key == "Enter" && clear()}}
          class="autocomplete-clear-button"
          >{@html clearText}</span>
      {/if}
    </div>
    <div
      class="{dropdownClassName ? dropdownClassName : ''} autocomplete-list {showList ? '' : 'hidden'}
      is-fullwidth"
      bind:this={list}
    >
      {#if filteredListItems && filteredListItems.length > 0}
          {@render dropdownHeader?.(filteredListItems.length, maxItemsToShowInList)}
  
          {#each filteredListItems as listItem, i}
              {#if listItem && (maxItemsToShowInList <= 0 || i < maxItemsToShowInList)}
                  <div
                      class="autocomplete-list-item"
                      class:selected={i === highlightIndex}
                      class:confirmed={isConfirmed(listItem.item)}
                      on:click={() => onListItemClick(listItem)}
                      on:keypress={(e) => {e.key == "Enter" && onListItemClick(listItem)}}
                      on:pointerenter={() => {
                          highlightIndex = i
                      }}
                  >
                  
                  {@render item?.(
                      listItem.item,
                      listItem.highlighted ? listItem.highlighted : listItem.label
                  )}
                  {#if !item}
                    {#if listItem.highlighted}
                        {@html listItem.highlighted}
                    {:else}
                        {@html listItem.label}
                    {/if}
                  {/if}
                  </div>
              {/if}
          {/each}
      
          
          {@render dropdownFooter?.(filteredListItems.length, maxItemsToShowInList)}
          {#if dropdownFooter}
              {#if maxItemsToShowInList > 0 && filteredListItems.length > maxItemsToShowInList}
                  <div class="autocomplete-list-item-no-results">
                      ...{filteredListItems.length - maxItemsToShowInList}
                      {moreItemsText}
                  </div>
              {/if}
          {/if}
      {:else if loading && loadingText}
          <div class="autocomplete-list-item-loading">
              {@render loadingSlot?.(loadingText)}
              {#if !loadingSlot}
                {loadingText}
              {/if}
          </div>
      
      {:else if create}
          <div
              class="autocomplete-list-item-create"
              on:click={selectItem}
              on:keypress={(e) => {e.key == "Enter" && selectItem()}}
          >
              {@render createSlot?.(createText)}
              {#if !createSlot}
                {createText}
              {/if}
          </div>
      
      {:else if noResultsText}
          <div class="autocomplete-list-item-no-results">
              {@render noResults?.(noResultsText)}
              {#if !noResults}
                {noResultsText}
              {/if}
          </div>
      {/if}
    </div>
  </div>
  
  <svelte:window on:click={onDocumentClick} on:scroll={() => setPositionOnNextUpdate = true} />
  
  <style>
    .autocomplete {
      min-width: 200px;
      display: inline-block;
      max-width: 100%;
      position: relative;
      vertical-align: top;
      height: 2.25em;
    }
  
    .autocomplete:not(.hide-arrow):not(.is-loading)::after {
      border: 3px solid;
      border-radius: 2px;
      border-right: 0;
      border-top: 0;
      content: " ";
      display: block;
      height: 0.625em;
      margin-top: -0.4375em;
      pointer-events: none;
      position: absolute;
      top: 50%;
      -webkit-transform: rotate(-45deg);
      transform: rotate(-45deg);
      -webkit-transform-origin: center;
      transform-origin: center;
      width: 0.625em;
      border-color: #3273dc;
      right: 1.125em;
      z-index: 4;
    }
  
    .autocomplete.show-clear:not(.hide-arrow)::after {
      right: 2.3em;
    }
  
    .autocomplete * {
      box-sizing: border-box;
    }
    .autocomplete-input {
      font: inherit;
      width: 100%;
      height: 100%;
      padding: 5px 11px;
    }
  
    .autocomplete:not(.hide-arrow) .autocomplete-input {
      padding-right: 2em;
    }
    .autocomplete.show-clear:not(.hide-arrow) .autocomplete-input {
      padding-right: 3.2em;
    }
    .autocomplete.hide-arrow.show-clear .autocomplete-input {
      padding-right: 2em;
    }
  
    .autocomplete-list {
      background: #fff;
      position: relative;
      width: 100%;
      overflow-y: auto;
      z-index: 99;
      padding: 10px 0;
      top: 0px;
      border: 1px solid #999;
      max-height: calc(15 * (1rem + 10px) + 15px);
      user-select: none;
    }
    .autocomplete-list:empty {
      padding: 0;
    }
    .autocomplete-list-item {
      padding: 5px 15px;
      color: #333;
      cursor: pointer;
      line-height: 1;
    }
  
    .autocomplete-list-item.confirmed {
      background-color: #789fed;
      color: #fff;
    }
    .autocomplete-list-item.selected {
      background-color: #2e69e2;
      color: #fff;
    }
    .autocomplete-list-item-no-results {
      padding: 5px 15px;
      color: #999;
      line-height: 1;
    }
    .autocomplete-list-item-create {
      padding: 5px 15px;
      line-height: 1;
    }
    .autocomplete-list-item-loading {
      padding: 5px 15px;
      line-height: 1;
    }
  
    .autocomplete-list.hidden {
      visibility: hidden;
    }
  
    .autocomplete.show-clear .autocomplete-clear-button {
      cursor: pointer;
      display: block;
      text-align: center;
      position: absolute;
      right: 0.1em;
      padding: 0.3em 0.6em;
      top: 50%;
      -webkit-transform: translateY(-50%);
      -ms-transform: translateY(-50%);
      transform: translateY(-50%);
      z-index: 4;
    }
  
    .autocomplete:not(.show-clear) .autocomplete-clear-button {
      display: none;
    }
  
    .autocomplete select {
      display: none;
    }
  
    .autocomplete.is-multiple .input-container {
      height: auto;
      box-shadow: inset 0 1px 2px rgba(10, 10, 10, 0.1);
      border-radius: 4px;
      border: 1px solid #b5b5b5;
      padding-left: 0.4em;
      padding-right: 0.4em;
      display: flex;
      flex-wrap: wrap;
      align-items: stretch;
      background-color: #fff;
    }
  
    .autocomplete.is-multiple .tag {
      display: flex;
      margin-top: 0.5em;
      margin-bottom: 0.3em;
    }
  
    .autocomplete.is-multiple .tag.is-delete {
      cursor: pointer;
    }
  
    .autocomplete.is-multiple .tags {
      margin-right: 0.3em;
      margin-bottom: 0;
    }
  
    .autocomplete.is-multiple .autocomplete-input {
      display: flex;
      width: 100%;
      flex: 1 1 50px;
      min-width: 3em;
      border: none;
      box-shadow: none;
      background: none;
    }
  </style>
