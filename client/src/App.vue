<template>

<Card />
</template>

<script>
import Card from "./components/card.vue";
import axios from "axios";

export default {
    name: "App",
    mounted() {
        axios
            .get("/api/owner")
            .then((result) => {
                console.log("Setting owner to " + result.data);
                this.owner = result.data;
            })
            .catch((err) => {
                console.error(err);
            });
    },
    components: {
        Card,
    },
    data: () => ({
        owner: "PlaceholderTitleForOwner",
    }),
};
</script>
