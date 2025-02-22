<script>
  import { getContext, onMount } from "svelte";
  import KanbanColumns from "./components/Columns.svelte";
  import SortColumns from "./components/ColumnsSort.svelte";
  import {
    Input,
    Label,
    Layout,
    ModalContent,
    Modal,
  } from "@budibase/bbui";

  // variables
  export let dataProvider;
  export let table;
  export let kanbanCardTitles;
  export let onClick;
  export let CardWidths;
  export let title;

  let tableStatuses = [];
  let columns = {};
  let droppable;
  let columnsLoaded = false;
  let kanbanColumns;
  let modal;
  let orderModal;
  let newColumnTitle;
  let columnOrder = 0;
  let updatedTableStatuses = {};
  let searchQuery = "";

  $: filteredColumns = searchQuery.length > 3
  ? Object.fromEntries(Object.entries(columns).map(([key, value]) => {
      const filteredCards = value.filter(card => {
        const regex = new RegExp(searchQuery, 'gi');
        const match = (card.Title?.match(regex) || []).length > 0 ||
          (card.Description?.match(regex) || []).length > 0;
        return match;
      });
      return [key, filteredCards];
    }))
  : columns;

  const cardsTableId = dataProvider?.datasource.tableId;
  const statusTableId = table?.tableId;

  const { styleable, API, notificationStore } = getContext("sdk");
  const component = getContext("component");
  // fetch cards table
  function fetchCardsTable() {
    return API.fetchTableData(cardsTableId)
      .then(function (data) {
        dataProvider.rows = data;
        return data;
      })
      .catch((err) => {
        console.log(err); // logging the error to the console.
      });
  }
  async function fetchTables() {
    try {
      await fetchCardsTable();
      const data = await API.fetchTableData(statusTableId);
      tableStatuses = data;
      if (tableStatuses.length > 0 && tableStatuses[0].hasOwnProperty('Order')) {
        tableStatuses.sort((a, b) => a.Order - b.Order);
      }
      columnOrder = data.length + 1;
      const newColumns = {};
      tableStatuses.forEach((status) => {
        newColumns[status.Title] = [];
      });
      dataProvider.rows.forEach((card) => {
        const state = card[kanbanCardTitles][0].primaryDisplay;
        newColumns[state].push(card);
      });
      columns = newColumns; // update columns object
    } catch(error){
      console.log("Unable to fetch cards table");
    } finally {
      columnsLoaded = true;
    }
}

  // on page load
  onMount(() => {
    // define a Promise that resolves when the variable is populated
    function waitForVariable(dataProvider) {
      return new Promise(function (resolve) {
        var checkVariable = function () {
          if (dataProvider) {
            resolve();
          } else {
            setTimeout(checkVariable, 100); // check variable every 100ms
          }
        };
        checkVariable();
      });
    }
    // wait for the variable to become populated, then run the process
    waitForVariable(dataProvider)
      .then(function () {
        fetchTables();
      })
      .catch(function (err) {
        console.log(err); // logging the error to the console.
      });
  });
  // add new column
  async function addColumn() {
    // check if column already exists, if it does don't let them create.
    if (newColumnTitle && !(newColumnTitle in columns)) {
      try {
        await API.saveRow({
          // create a new column on the backend
          tableId: statusTableId,
          Title: newColumnTitle,
          Order: columnOrder,
        });
        await fetchTables(); // reload main data after creating new column row
      } catch (error) {
        console.log(error);
      }
      notificationStore.actions.success(
        `New column: ${newColumnTitle} has been successfully created!`
      );
    } else {
      notificationStore.actions.error(
        "Sorry you have either tried to add a row without a Title, or tried to create a duplicate column."
      );
    }
  }
  function handleTableStatusesUpdated(event) {
    updatedTableStatuses = event.detail;
  }
  async function refreshColumns() {
    try {
      const promises = updatedTableStatuses.map((status, index) => {
          return API.saveRow({
              ...status,
              Order: index + 1,
          }).catch((error) => {
              console.error(error);
          });
      });
      await Promise.all(promises);
      await fetchTables();
      notificationStore.actions.success(
          `Your column orders have been successfully rearranged.`
      );
      await fetchTables();
    } catch (error) {
    }
  }
  // call the moveItem() function in the child component
  function handleDrop(event) {
    childComponent.moveItem();
  }
</script>
<div use:styleable={$component.styles} class="container">
  {#if !dataProvider || !table}
    <div class="placeholder">
      Please provide a Data Provider and a Relationship Table
    </div>
  {:else if columnsLoaded}
    <div class="header">
        <h2>{#if title}{title}{/if}</h2>
        <div class="header">
          <input type="text" bind:value={searchQuery} placeholder="Search..." class="customInput spectrum-Textfield-input" />
          <button
            on:click={modal.show()}
            class="spectrum-Button spectrum-Button--sizeM spectrum-Button--cta"
            style="margin-right: 0.5rem; min-width: fit-content;"
            >Add Column</button
          >
          {#if tableStatuses.length > 0 && tableStatuses[0].hasOwnProperty('Order')}
            <button style="border: none; cursor: pointer; background-color: transparent;" on:click={orderModal.show()}>
              <svg xmlns="http://www.w3.org/2000/svg" height="18" viewBox="0 0 18 18" width="18" class="customSvgColour"><rect id="Canvas" fill="fillCurrent" opacity="0" width="18" height="18" /><path class="fill" d="M16.45,7.8965H14.8945a5.97644,5.97644,0,0,0-.921-2.2535L15.076,4.54a.55.55,0,0,0,.00219-.77781L15.076,3.76l-.8365-.836a.55.55,0,0,0-.77781-.00219L13.4595,2.924,12.357,4.0265a5.96235,5.96235,0,0,0-2.2535-.9205V1.55a.55.55,0,0,0-.55-.55H8.45a.55.55,0,0,0-.55.55V3.106a5.96235,5.96235,0,0,0-2.2535.9205l-1.1-1.1025a.55.55,0,0,0-.77781-.00219L3.7665,2.924,2.924,3.76a.55.55,0,0,0-.00219.77781L2.924,4.54,4.0265,5.643a5.97644,5.97644,0,0,0-.921,2.2535H1.55a.55.55,0,0,0-.55.55V9.55a.55.55,0,0,0,.55.55H3.1055a5.967,5.967,0,0,0,.921,2.2535L2.924,13.4595a.55.55,0,0,0-.00219.77782l.00219.00218.8365.8365a.55.55,0,0,0,.77781.00219L4.5405,15.076,5.643,13.9735a5.96235,5.96235,0,0,0,2.2535.9205V16.45a.55.55,0,0,0,.55.55H9.55a.55.55,0,0,0,.55-.55V14.894a5.96235,5.96235,0,0,0,2.2535-.9205L13.456,15.076a.55.55,0,0,0,.77782.00219L14.236,15.076l.8365-.8365a.55.55,0,0,0,.00219-.77781l-.00219-.00219L13.97,12.357a5.967,5.967,0,0,0,.921-2.2535H16.45a.55.55,0,0,0,.55-.55V8.45a.55.55,0,0,0-.54649-.55349ZM11.207,9A2.207,2.207,0,1,1,9,6.793H9A2.207,2.207,0,0,1,11.207,9Z" /></svg>
            </button>
          {/if}
        </div>
    </div>
    <div
      class="kanban-board"
      on:drop|preventDefault={() => handleDrop}
      bind:this={droppable}
    >
      <KanbanColumns
        columns={filteredColumns}
        {droppable}
        {tableStatuses}
        {kanbanCardTitles}
        {onClick}
        {cardsTableId}
        {CardWidths}
        {fetchTables}
        bind:this={kanbanColumns}
      />
    </div>
    <p>You have a total of {Object.keys(columns).length} columns</p>
    <Modal bind:this={orderModal}>
      <ModalContent title="Columns Positions" size="XL" onConfirm={refreshColumns}>
        <SortColumns
          {tableStatuses}
          on:tableStatusesUpdated={handleTableStatusesUpdated}
        />
      </ModalContent>
    </Modal>
  {:else}
    <span class="spinner spinner-large"></span>
  {/if}
</div>
<style>
  *:focus {
    outline: none;
  }
  .kanban-board {
    display: flex;
    overflow: scroll;
    -ms-overflow-style: none; /* IE and Edge */
    scrollbar-width: none; /* Firefox */
    width: 100%;
  }
  .kanban-board::-webkit-scrollbar {
    display: none;
  }
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin: 0.5rem;
  }
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
  .customInput {
    display: block;
    border-radius: 4px;
    /* border: 1px solid rgb(220 220 220); */
    padding: 6px;
    margin-right: 0.5rem;
  }
  .spinner {
    width: 1.5rem;
    height: 1.5rem;
    border-top-color: #444;
    border-left-color: #444;
    animation: spinner 400ms linear infinite;
    border-bottom-color: transparent;
    border-right-color: transparent;
    border-style: solid;
    border-width: 2px;
    border-radius: 50%;  
    box-sizing: border-box;
    display: inline-block;
    vertical-align: middle;
  }
  .spinner-large {
    width: 5rem;
    height: 5rem;
    border-width: 6px;
  }
  @keyframes spinner {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  .customSvgColour {
    fill: var(--spectrum-global-color-gray-900);
  }
</style>