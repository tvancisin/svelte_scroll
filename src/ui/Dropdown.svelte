<script>
    import { createEventDispatcher } from "svelte";

    const dispatch = createEventDispatcher();
    export let selectedCountry = "Russia";

    function handleSelection(country) {
        selectedCountry = country;
        dispatch("close", selectedCountry);
    }
</script>

<div class="dropdown">
    <button class="dropbtn">
        <i class="fa fa-globe"></i>
    </button>
    <div class="dropdown-content">
            <a href="#" on:click={() => handleSelection("Russia")}>Russia</a>
            <a href="#" on:click={() => handleSelection("China")}>China</a>
    </div>
</div>


<style>
    /* Container needed to position the dropdown content */
    .dropdown {
        position: absolute;
        top: 1px;
        right: 60px;
        display: inline-block;
        z-index: 10;
        font-family: "Montserrat";
    }

    @media only screen and (max-width: 500px) {
        .dropdown {
            right: 40px;
        }
    }

    .dropbtn {
        position: absolute;
        background-color: white;
        border: 1px solid rgb(173, 173, 173);
        color: black;
        padding: 5px 16px;
        text-align: center;
        text-decoration: none;
        border-radius: 2px;
        display: inline-block;
        font-size: 1em;
        cursor: pointer;
        z-index: 10;
    }

    .dropbtn:hover {
        background-color: #aa4197;
        color: white;
    }

    @media only screen and (max-width: 768px) {
        .dropbtn {
            font-size: 0.7em;
        }
    }

    @media only screen and (max-width: 500px) {
        .dropbtn {
            padding: 4px 10px;
            font-size: 0.53em;
        }
    }
    /* Dropdown content (hidden by default) */
    .dropdown-content {
        display: none;
        position: absolute;
        top: 30px;
        right:-52px;
        background-color: #f9f9f9;
        min-width: 150px;
        box-shadow: 0px 8px 16px 0px rgba(0, 0, 0, 0.2);
        z-index: 1;
        max-height: 350px;
        overflow-y: auto;
    }

    @media only screen and (max-width: 500px) {
        .dropdown-content {
            right: -35px;
            top: 25px;
        }
    }

    /* Links inside the dropdown */
    .dropdown-content a {
        color: black;
        padding: 12px 16px;
        text-decoration: none;
        display: block;
    }

    /* Change color of dropdown links on hover */
    .dropdown-content a:hover {
        background-color: #f1f1f1;
    }

    /* Show the dropdown menu on hover */
    .dropdown:hover .dropdown-content {
        display: block;
    }

    /* Change the background color of the dropdown button when the dropdown content is shown */
    .dropdown:hover .dropbtn {
        background-color: #aa4197;
        color: white;
    }
</style>
