<template>
    <div class="login">

        <!-- No webId  -->
        <div class="form" v-if="!this.login">
            <h2>Please select your identity provider :</h2>
            <el-select v-model="selectValue" placeholder="please select..." @change="selectProvider">
                <el-option
                    v-for="item in options"
                    :key="item.value"
                    :label="item.label"
                    :value="item.value">
                </el-option>
            </el-select>
            <el-button @click="submit" class="submitButton" type="info" plain>Submit</el-button>
        </div>

        <!-- Has webId  -->
        <div v-else>

            <!-- WebId  -->
            <h2>Your webId is : {{ this.webId }}</h2>

            <!-- Username  -->
            <h2 v-if="this.profile !== null">Your name is : {{ this.userName }}</h2>

            <!-- Notepad  -->
            <div v-if="this.notePad !== null" class="notepad">
                <h2>{{ this.userName }}'s NotePad</h2>
                <el-input placeholder="Add note..." v-model="newNote">
                    <el-button @click="addNote" slot="append" icon="el-icon-edit"></el-button>
                </el-input>
                <h2 class="notes" v-for="note in notes" :key="note">{{ note }}</h2>
            </div>

            <!-- Get Name button  -->
            <el-button @click="getName" class="" type="general" plain>Get Your Profile</el-button>

            <!-- Get Notepad button  -->
            <el-button v-if="this.profile !== null" @click="getNotesList" class="" type="general" plain>Get Note List
            </el-button>

            <!-- Logout button -->
            <el-button @click="logout" class="submitButton" type="danger" plain>Log out</el-button>
        </div>

    </div>
</template>

<script>
import auth from "solid-auth-client";
import {fetchDocument, createDocument} from 'tripledoc';
import {foaf, solid, schema, space, rdf} from 'rdf-namespaces';

export default {
    name: "Solid",
    data() {
        return {
            login: false,
            // IDP options
            options: [{
                value: 'https://inrupt.net',
                label: 'Inrupt'
            }, {
                value: 'https://solid.community',
                label: 'Solid Community'
            }],
            // Select IDP
            selectValue: "",
            // Your webId
            webId: "",
            // Your user name
            userName: "",
            // Profile object
            profile: null,
            // NotePad object
            notePad: null,
            // Input value of new note
            newNote: "",
            // Notes from your pod (only text)
            notes: null
        }
    },
    methods: {
        // Event listener : selector on change
        selectProvider(event) {
            this.selectValue = event;
        },
        // Submit IDP selector form & login user
        submit() {
            auth.login(this.selectValue).then(webId => {
                this.webId = webId;
                this.login = true;
            });
        },
        // Get webId by looking up session in localstorage
        async getWebId() {
            let session = await auth.currentSession();
            if (session) {
                return session.webId;
            } else {
                return '';
            }
        },
        // Get user name
        async getName() {
            let webIdDoc = await fetchDocument(this.webId);
            let profile = webIdDoc.getSubject(this.webId);
            this.profile = profile;
            this.userName = profile.getString(foaf.name);
        },
        async getNotesList() {
            /* 1. Check if a Document tracking our notes already exists. */
            let publicTypeIndexRef = this.profile.getRef(solid.publicTypeIndex);
            let publicTypeIndex = await fetchDocument(publicTypeIndexRef);
            let notesListEntry = publicTypeIndex.findSubject(solid.forClass, schema.TextDigitalDocument);
            let noteslist = null;

            /* 2. If it doesn't exist, create it. */
            if (notesListEntry === null) {
                // We will define this function later:
                noteslist = this.initialiseNotesList(this.profile, publicTypeIndex);
                this.notePad = noteslist;
                return noteslist;
            }

            /* 3. If it does exist, fetch that Document. */
            let notesListRef = notesListEntry.getRef(solid.instance);
            noteslist = await fetchDocument(notesListRef);
            this.notePad = noteslist;
            this.notes = noteslist.getAllSubjectsOfType(schema.TextDigitalDocument).map(function (note) {
                return note.getString(schema.text);
            });
            return noteslist;
        },
        // Creates anew document that contain the notes, and adds it to the Public Type Index.
        async initialiseNotesList(profile, typeIndex) {
            // Get the root URL of the user's Pod:
            let storage = profile.getRef(space.storage);

            // Decide at what URL within the user's Pod the new Document should be stored:
            let notesListRef = storage + 'public/notes.ttl';
            // Create the new Document:
            let notesList = createDocument(notesListRef);
            await notesList.save();

            // Store a reference to that Document in the public Type Index for `schema:TextDigitalDocument`:
            let typeRegistration = typeIndex.addSubject();
            typeRegistration.addRef(rdf.type, solid.TypeRegistration)
            typeRegistration.addRef(solid.instance, notesList.asRef())
            typeRegistration.addRef(solid.forClass, schema.TextDigitalDocument)
            await typeIndex.save([typeRegistration]);

            // And finally, return our newly created (currently empty) notes Document:
            return notesList;
        },
        async addNote() {
            // Initialise the new Subject:
            let newNote = this.notePad.addSubject();

            // Indicate that the Subject is a schema:TextDigitalDocument:
            newNote.addRef(rdf.type, schema.TextDigitalDocument);

            // Set the Subject's `schema:text` to the actual note contents:
            newNote.addString(schema.text, this.newNote);

            // Store the date the note was created (i.e. now):
            newNote.addDateTime(schema.dateCreated, new Date(Date.now()))

            let success = await this.notePad.save([newNote]);
            this.notes.push(this.newNote);
            return success;
        },
        // Log out user
        logout() {
            auth.logout().then(() => console.log("Goodbye!"));
            this.login = false;
        }
    },
    mounted() {
        this.getWebId().then(webID => {
            if (webID !== '') {
                this.login = true;
            }
            this.webId = webID;
        })
    }
}
</script>

<style scoped>
.login {
    height: 100%;
    display: grid;
    place-items: center;
}

.submitButton {
    margin-left: 20px;
}

.notepad {
    display: block;
    height: 100%;
    width: 100%;
    margin-bottom: 30px;
    padding-top: 10px;
    padding-bottom: 50px;
    background: bisque;
    border-radius: 20px;
}

.notes {
    background: orange;
    text-align: center;
    width: 70%;
    padding-top: 15px;
    padding-bottom: 15px;
    margin: 30px auto auto;
}

.el-input {
    width: 70%;
}
</style>