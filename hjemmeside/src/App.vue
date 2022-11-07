<script setup>
import { ref } from "vue";
import { reactive } from "vue";
import CircleProgress from "vue3-circle-progress";
import EasyDataTable from "vue3-easy-data-table";
import vSelect from "vue-select";
import Datepicker from "@vuepic/vue-datepicker";
import "@vuepic/vue-datepicker/dist/main.css";
import * as Realm from "realm-web";

// Defines DB Connection
new Realm.App({ id: "App ID" });
const app = Realm.getApp("App ID");

//#region AnonymousLogin
// At start of App, logs user on anonymously to allow access to DB
async function loginAnonymous() {
  // Create an anonymous credential
  const credentials = Realm.Credentials.anonymous();
  try {
    // Authenticate the user
    const anonymousUser = await app.logIn(credentials);
    // `App.currentUser` updates to match the logged in user
    console.assert(anonymousUser.id === app.currentUser.id);
    return anonymousUser;
  } catch (err) {
    console.error("Failed to log in", err);
  }
}
//#endregion

//#region DB Variables

// Get DB and all collections
let mongodb
let usersC
let logsC
let teamsC
let classesC

async function setupLoginDB() {
  const anonymousUser = await loginAnonymous();
  mongodb = app.currentUser.mongoClient("mongodb-atlas");
  usersC = mongodb.db("vue-db1").collection("users");
  logsC = mongodb.db("vue-db1").collection("stepLogs");
  teamsC = mongodb.db("vue-db1").collection("teams");
  classesC = mongodb.db("vue-db1").collection("classes");
}
setupLoginDB();
//#endregion
//#region In App Variables
const now = new Date();

// Menu Open / 1 = logon / 2 = steps add / 3 = settings / 4 = createUser / 5 = viewLogs / 6 = deleteUser / 7 = confirmDelete
let screenOpenStatus = ref(0);
let userLogged = ref(false);
const disabledItems = ref(false);

// Ref on variable updates all places where it is used, ex in a calculation
let user = ref("");
let username = ref("");
let email = ref("");
let password = ref("");
let _class = ref("");
let team = ref("");
let _steps = ref("");
let description = ref("");
let timeStampByUser = ref("");
const error = ref("No");

let tmpCalcToday = ref(0);
let userStepsToday = ref(0);

let _teams = ref(["team1", "team2", "team3", "team4", "team5"]);
let _classes = ref([
  "S1",
  "S2",
  "S3",
  "V1",
  "V2",
  "V3",
  "T1",
  "T2",
  "T3",
  "AspitLab",
  "Personale",
]);

let header = ref([
  { text: "Navn", value: "name" },
  { text: "Skridt", value: "steps" },
  { text: "Klasse", value: "class" },
  { text: "hold", value: "team" },
]);
let headerTeam = ref([
  { text: "Navn", value: "teamName" },
  { text: "Skridt", value: "steps" },
]);
let headerClass = ref([
  { text: "Navn", value: "className" },
  { text: "Skridt", value: "steps" },
]);
let headerLogs = ref([
  { text: "Skridt", value: "steps" },
  { text: "Beskrivelse", value: "description" },
  { text: "Dato", value: "timeStampByUser" },
]);

// Reactive in a variable is used when updating Objects or lists of items to update all nested properties
let loggedUser = reactive([1]);
let itemTop5 = reactive([5]);
let itemClass = reactive([5]);
let itemTeam = reactive([5]);
let itemTopClass = reactive([5]);
let itemTopTeam = reactive([5]);
let itemLogs = reactive([6]);

let popupVisible = ref(false);
let popupContent = ref("");
//#endregion

//#region Popup Func
function closePopup() {
  popupContent.value = "";
  popupVisible.value = false;
}

function openPopup(tmpText) {
  popupContent.value = tmpText;
  popupVisible.value = true;
}
//#endregion

//#region Create Account / Delete Account / Logon / Logout / Create new StepLog
const createAccountEmailPassword = async () => {
  try {
    if (team.value == "") {
      team.value = null;
    }
    // opensPopup if credentials are invalid else it runs the code as normal
    if (username.value == "") {
      openPopup("Ugyldigt brugernavn");
    } else if (email.value == "") {
      openPopup("Ugyldig Email");
    } else if (password.value == "" || password < 6) {
      openPopup("Ugyldigt kodeord, skal være mindst 6 tegn");
    } else if (
      _class.value == null ||
      _class.value == undefined ||
      _class.value == ""
    ) {
      openPopup("Klasse felt skal udfyldes");
    } else {
      // Registers a new user in the DB using email and password
      await app.emailPasswordAuth.registerUser(email.value, password.value);

      // Sets the value of the item in personal stats table to our logged user to display statistics
      let tmpTeam = team;
      if (tmpTeam.value == null) {
        tmpTeam = "Intet Hold";
      }
      loggedUser[0] = {
        name: username.value,
        steps: Number(0),
        class: _class.value,
        team: tmpTeam.value,
      };

      // Calls login function to login to the newly created user
      await loginEmailPassword(
        Realm.Credentials.emailPassword(email.value, password.value)
      );

      // Inserts the user document into the DB in the users collection(usersC)
      const resp = await usersC.insertOne({
        userID: app.currentUser.id,
        username: username.value,
        class: _class.value,
        team: team.value,
        steps: Number(0),
      });
    }
  } catch (err) {
    console.error("Failed to Log In", err);
    error.value = err;
  }
};

const loginEmailPassword = async () => {
  // Returns a credentials instance that the DB can process
  const credentials = Realm.Credentials.emailPassword(
    email.value,
    password.value
  );

  try {
    // opensPopup if credentials are invalid else it runs the code as normal
    if (email.value == "") {
      openPopup("Ugyldig Email");
    } else if (password.value == "" || password.value < 6) {
      openPopup("Ugyldigt kodeord, skal være mindst 6 tegn");
    } else {
      // Logs our user in
      user = await app.logIn(credentials);
      console.assert(user.id === app.currentUser.id);

      userLogged.value = true;
      screenOpenStatus.value = 0;

      // Gets our newly logged user's Steps Count to add to the progress circle count
      const dailySteps = await logsC.find({ logUserRefID: user.id });
      for (let i = 0; i < dailySteps.length; i++) {
        let tmpDate = new Date(dailySteps[i].timeStampByUser);
        if (
          tmpDate.getDate() == now.getDate() &&
          tmpDate.getMonth() == now.getMonth() &&
          tmpDate.getFullYear() == now.getFullYear()
        ) {
          userStepsToday.value += dailySteps[i].steps;
        }
      }

      // Sets the value of the item in personal stats table to our logged user to display statistics
      const tmpUser = await usersC.findOne({ userID: user.id });
      if (tmpUser != null) {
        loggedUser[0] = {
          name: tmpUser.username,
          steps: tmpUser.steps,
          class: tmpUser.class,
          team: tmpUser.team,
        };
      }
      calcPercentageToday();
      updateData();
    }
  } catch (err) {
    console.error("Failed to Log In", err);
    error.value = err;
    openPopup("Ugyldig email eller password, Venligst prøv igen.");
  }
};

// Self Explanatory
const userLogout = async () => {
  // If there is a currentUser then logOut
  await app.currentUser?.logOut();
  userStepsToday.value = 0;
  userLogged.value = false;
  user = "";
  loggedUser[0] = null;
};

const deleteAccount = async () => {
  try {
    // opensPopup if credentials are invalid else it runs the code as normal
    if (email.value == "") {
      openPopup("Ugyldig Email");
    } else if (password.value == "" || password.value < 6) {
      openPopup("Ugyldigt kodeord, skal være mindst 6 tegn");
    } else {
      let tmpUserDelete = usersC.findOne({ username: username.value })
      if (
        tmpUserDelete != null &&
        app.currentUser.id == tmpUserDelete.userID
      ) {
        // Updates the team the user is part of to remove their steps
        tmpUser = await usersC.findOne({ userID: app.currentUser.id });
        // Checks if user is part of a team since the value is nullable
        if (tmpUser.team != null) {
          let userTeam = await teamsC.findOne({ teamName: tmpUser.team });
          let stepCalc = await userTeam.steps - Number(app.currentUser.steps);
          resp = await teamsC.updateOne(
            { teamName: tmpUser.team },
            { $set: { steps: stepCalc } }
          );
        }

        // Updates the class the user is part of to remove their steps
        let userClass = await classesC.findOne({ className: tmpUser.class });
        let stepCalc = await userClass.steps - Number(app.currentUser.steps);
        resp = await classesC.updateOne(
          { className: tmpUser.class },
          { $set: { steps: stepCalc } }
        );

        await app.deleteUser(tmpUserDelete);
        // anonymousUser = loginAnonymous();
      }
      else {
        console.log("username and email invalid")
      }
    }
  } catch (err) {
    console.error("Failed to delete account", err);
    error.value = err;
    openPopup("Ugyldig email eller password, Venligst prøv igen.");
  }
};

const createNewStepsLog = async () => {
  try {
    // opensPopup if credentials are invalid else it runs the code as normal
    if (Number(_steps.value) == 0 || _steps == String) {
      openPopup("Kan ikke tilføje ny antal skridt til brugeren hvis det er 0");
    } else if (timeStampByUser.value == null) {
      openPopup("Venligst indtast en dato");
    } else {
      // Awaits Response to insert an new log into the stepLogs(logsC) Collection
      let resp = await logsC.insertOne({
        logUserRefID: app.currentUser.id,
        steps: Number(_steps.value),
        description: description.value,
        timeStampByUser: timeStampByUser.value,
        timeStamp: new Date(),
      });

      // Updates the current user steps to match the newly added steps including their already existing
      let tmpUser = await usersC.findOne({ userID: app.currentUser.id });
      let tmpSteps = Number(tmpUser.steps) + Number(_steps.value);
      resp = await usersC.updateOne(
        { userID: app.currentUser.id },
        { $set: { steps: tmpSteps } }
      );

      // Updates the team that the current user is a part of to include the newly added steps
      tmpUser = await usersC.findOne({ userID: app.currentUser.id });
      // Checks if user is part of a team since the value is nullable
      if (tmpUser.team != null) {
        let userTeam = await teamsC.findOne({ teamName: tmpUser.team });
        let stepCalc = userTeam.steps + Number(_steps.value);
        resp = await teamsC.updateOne(
          { teamName: tmpUser.team },
          { $set: { steps: stepCalc } }
        );
      }

      // Updates the class that the current user is a part of to include the newly added steps
      let userClass = await classesC.findOne({ className: tmpUser.class });
      let stepCalc = userClass.steps + Number(_steps.value);
      resp = await classesC.updateOne(
        { className: tmpUser.class },
        { $set: { steps: stepCalc } }
      );

      // Sets the variable for progress circle to include newly added steps IF it was added today
      let tmpTime = new Date(timeStampByUser.value);
      if (tmpTime.getDay() == now.getDay()) {
        userStepsToday.value += Number(_steps.value);
      }

      calcPercentageToday();
      menuScreenToggle(0);
      updateData();
    }
  } catch (err) {
    console.error("Failed to create new log", err);
    error.value = err;
  }
};
//#endregion

//#region Toggle Logon/Step/CreateAccount-Screens
function menuScreenToggle(num) {
  screenOpenStatus.value = num;
  if ((num = 1 || num == 4 || num == 6)) {
    resetInputLogon();
  }
  if ((num = 2)) {
    resetInputSteps();
  }
}

const viewLogsScreenToggle = async (num) => {
  menuScreenToggle(num);
  let yourLogs = await logsC.find({ logUserRefID: app.currentUser.id });

  for (let i = 0; i < yourLogs.length; i++) {
    itemLogs[i] = {
      steps: yourLogs[i].steps,
      description: yourLogs[i].description,
      timeStampByUser: yourLogs[i].timeStampByUser.toDateString(),
    };
  }
};

const resetPassword = async () => {
  let email = "";
  await app.emailPasswordAuth.sendResetPasswordEmail({ email });
};
//#endregion

//#region ResetInput / Calc todays steps
function resetInputLogon() {
  username.value = "";
  email.value = "";
  password.value = "";
  _class.value = "";
  team.value = "";
}

function resetInputSteps() {
  _steps.value = 0;
  description.value = "";
  timeStampByUser.value = "";
}

const calcPercentageToday = async () => {
  // function for getting users steps

  if (userStepsToday.value < 13000) {
    tmpCalcToday.value = userStepsToday.value / 130;
  } else {
    tmpCalcToday.value = 100;
  }

  tmpCalcToday.value = Math.floor(tmpCalcToday.value);
};
//#endregion

//#region Timed Data Update
const updateData = async () => {
  //#region Gets top 5 users of different collections depending on amount of steps
  const sortTop5User = await usersC.aggregate([
    { $sort: { steps: -1 } },
    { $limit: 5 },
  ]);
  const sortTop5Class = await classesC.aggregate([
    { $sort: { steps: -1 } },
    { $limit: 5 },
  ]);
  const sortTop5Team = await teamsC.aggregate([
    { $sort: { steps: -1 } },
    { $limit: 5 },
  ]);
  //#endregion

  // Checks if there is a user logged and if its not anonymous
  if (app.currentUser.customData.username != null) {
    // Sorts the top5 list from most to least steps
    let tmpUser = await usersC.findOne({ userID: app.currentUser.id });
    let yourClassMembers = await usersC.find({ class: tmpUser.class });
    yourClassMembers.sort((a, b) => b.steps - a.steps);

    // Updates our currently logged users statistics
    loggedUser[0] = {
      name: loggedUser[0].name,
      steps: tmpUser.steps,
      class: loggedUser[0].class,
      team: loggedUser[0].team,
    };

    // Sets the table containing your class members to update the content
    for (let i = 0; i < yourClassMembers.length; i++) {
      let tmpTeam = yourClassMembers[i].team;
      if (yourClassMembers[i].team == null) {
        tmpTeam = "Intet Hold";
      }
      itemClass[i] = {
        name: yourClassMembers[i].username,
        steps: yourClassMembers[i].steps,
        class: yourClassMembers[i].class,
        team: tmpTeam,
      };
    }

    // Sets the table containing your team members to update the content
    if (tmpUser.team != null) {
      let yourTeamMembers = await usersC.find({ team: tmpUser.team });
      yourTeamMembers.sort((a, b) => b.steps - a.steps);

      for (let i = 0; i < yourTeamMembers.length; i++) {
        itemTeam[i] = {
          name: yourTeamMembers[i].username,
          steps: yourTeamMembers[i].steps,
          class: yourTeamMembers[i].class,
          team: yourTeamMembers[i].team,
        };
      }
    }
  }

  for (let i = 0; i < 5; i++) {
    try {
      // Top 5 Users
      if (sortTop5User[i] == null) {
        break;
      }
      let tmpTeam = sortTop5User[i].team;
      if (sortTop5User[i].team == null) {
        tmpTeam = "Intet Hold";
      }
      itemTop5[i] = {
        name: sortTop5User[i].username,
        steps: sortTop5User[i].steps,
        class: sortTop5User[i].class,
        team: tmpTeam,
      };
    } catch (err) {
      console.error("Failed to get top 5 users", err);
      error.value = err;
    }
  }

  for (let i = 0; i < 5; i++) {
    try {
      // Top 5 Classes
      itemTopClass[i] = {
        className: sortTop5Class[i].className,
        steps: sortTop5Class[i].steps,
      };
    } catch (err) {
      console.error("Failed to get top 5 classes", err);
      error.value = err;
    }
  }
  
  for (let i = 0; i < 5; i++) {
    try {
      // Top 5 Teams
      itemTopTeam[i] = {
        teamName: sortTop5Team[i].teamName,
        steps: sortTop5Team[i].steps,
      };
    } catch (err) {
      console.error("Failed to get top 5 teams", err);
      error.value = err;
    }
  }
};

async function timer() {
  updateData();
  const myTimeout = setInterval(updateData, 10000);
}

timer();
//#endregion
</script>

<template>
  <div class="app" :class="{ inMenu: screenOpenStatus > 0 }">
    <div class="popupBlur" v-if="popupVisible">
      <div class="popup">
        <div class="popupText">{{ popupContent }}</div>
        <button class="popupBtn" @click="closePopup">Ok</button>
      </div>
    </div>
    <div class="headerNav">
      <div class="headerLogoContainer">
        <img class="headerLogo" src="../src/assets/logoAspit.png" />
        <p class="headerTitle">TælSkridt</p>
      </div>

      <div class="headerBtnContainer">
        <button v-if="!userLogged" class="btn headerBtn" @click="menuScreenToggle(1)" :disabled="screenOpenStatus > 0">
          Log ind
        </button>
        <button v-else class="btn headerBtn" @click="userLogout" :disabled="screenOpenStatus > 0">
          Log Af
        </button>

        <button class="btn headerBtn" @click="menuScreenToggle(2)" :disabled="screenOpenStatus > 0 || !userLogged">
          Tilføj Skridt
        </button>
        <button class="btn headerBtn settingsBtn" @click="menuScreenToggle(3)" :disabled="screenOpenStatus > 0">
          =
        </button>
      </div>
    </div>

    <div class="body">
      <div class="mainStarter">
        <!-- v-if="loggedOnUser != null" -->
        <div class="mainContainer personalSteps" v-if="userLogged">
          <span class="circleProgressFlex">
            <circle-progress :percent="tmpCalcToday" fill-color="#40bf6e" />
            <span class="circleProgressPercent">
              <p class="circleProgressNumber">{{ tmpCalcToday }}</p>
              <p>%</p>
            </span>
          </span>

          <div class="circleProgressTodaySteps">
            <p class="personalStepsP">Dagligt Mål</p>
            <p class="personalStepsP">Ud af 13000 Skridt</p>
          </div>
        </div>

        <div class="mainContainer personalStats" v-if="userLogged">
          <p class="personalP">Personlige Statistik</p>
          <easy-data-table :headers="header" :items="loggedUser" hide-footer alternating rows-per-page="5" />
        </div>

        <div class="mainContainer top5People">
          <p class="top5P">Top 5</p>
          <easy-data-table :headers="header" :items="itemTop5" hide-footer alternating rows-per-page="5" />
        </div>
        <div class="mainContainer yourTeam" v-if="userLogged && loggedUser[0].team != null">
          <p class="teamP">Dit Hold</p>
          <easy-data-table :headers="header" :items="itemTeam" hide-footer alternating />
        </div>
        <div class="mainContainer yourClass" v-if="userLogged">
          <p class="classP">Din Klasse</p>
          <easy-data-table :headers="header" :items="itemClass" alternating hide-footer />
        </div>

        <div class="mainContainer topTeam">
          <p class="classP">Top Hold</p>
          <easy-data-table :headers="headerTeam" :items="itemTopTeam" alternating hide-footer rows-per-page="5" />
        </div>

        <div class="mainContainer topClass">
          <p class="classP">Top Klasser</p>
          <easy-data-table :headers="headerClass" :items="itemTopClass" alternating hide-footer rows-per-page="5" />
        </div>
      </div>

      <div class="logon_Steps_Blur" v-if="screenOpenStatus > 0">
        <div class="logon_Steps_Container" :class="{
          logonContainerSize: screenOpenStatus != 5,
          logsScreenClass: screenOpenStatus == 5,
        }" v-if="screenOpenStatus > 0">
          <div class="addSteps" v-if="screenOpenStatus == 2">
            <button class="btn returnBtn" @click="menuScreenToggle(0)">
              X
            </button>
            <span class="inputBox inputSteps">
              <p class="p">Skridt Antal</p>
              <input class="textAddStepsAmount" v-model="_steps" />
            </span>
            <span class="inputBox inputDesc">
              <span class="addStepsDescContainer">
                <p class="p">Beskrivelse</p>
                <p class="p optionalFont">valgfri</p>
              </span>
              <input class="textAddStepsDesc" v-model="description" />
            </span>
            <span>
              <Datepicker v-model="timeStampByUser" class="datePick"></Datepicker>
            </span>
            <span class="addStepsBtnContainer">
              <button class="btn addStepsBtn" @click="createNewStepsLog">
                Tilføj
              </button>
            </span>
          </div>
          <div v-if="screenOpenStatus == 3">
            <button class="btn returnBtn" @click="menuScreenToggle(0)">
              X
            </button>
            <span class="settingsMenu">
              <button @click="viewLogsScreenToggle(5)" class="viewLogsBtn btn" :disabled="!userLogged">
                Dine Skridt
              </button>
              <button v-if="disabledItems" @click="menuScreenToggle(6)" class="deleteBtn btn" :disabled="!userLogged">
                Slet Bruger
              </button>
            </span>
          </div>
          <div v-if="screenOpenStatus == 5">
            <button class="btn returnBtn" @click="viewLogsScreenToggle(3)">
              X
            </button>
            <h2>Dine Skridt</h2>
            <div class="viewLogsWrapper">
              <easy-data-table class="viewLogs" :headers="headerLogs" :items="itemLogs" alternating hide-footer />
            </div>
          </div>
          <div v-if="screenOpenStatus == 6 || screenOpenStatus == 7">
            <button class="btn returnBtn" @click="menuScreenToggle(3)">
              X
            </button>
            <h3>Slet Bruger</h3>
            <span class="inputBox inputEmailDelete">
              <p class="p">Email</p>
              <input class="textDeleteEmail" type="email" v-model="email" />
            </span>
            <span class="inputBox inputUsernameDelete">
              <p class="p">Brugernavn</p>
              <input class="textDeleteUsername" type="username" v-model="username" />
            </span>
            <div class="deleteUserBtnContainer">
              <button class="btn btnDeleteUser" @click="menuScreenToggle(7)">
                Slet
              </button>
            </div>

            <div v-if="screenOpenStatus == 7" class="confirmDeleteContainer">
              <button class="btn confirmBtnDelete" @click="deleteAccount()">
                Slet Bruger
              </button>
              <h5 class="confirmP">You Sure?</h5>
              <button class="btn returnBtnDelete" @click="menuScreenToggle(6)">
                Annuller
              </button>
            </div>
          </div>
          <div class="logon" v-if="screenOpenStatus == 1 || screenOpenStatus == 4">
            <div class="logonToUser" v-if="screenOpenStatus == 1">
              <button class="btn returnBtn" @click="menuScreenToggle(0)">
                X
              </button>

              <span class="inputBox inputNameLogon">
                <p class="p">Email</p>
                <input class="textLogonName" type="email" v-model="email" />
              </span>
              <span class="inputBox inputPasswordLogon">
                <p class="p">Password</p>
                <input class="textLogonPassword" type="password" v-model="password" />
              </span>
              <a v-if="disabledItems" class="passwordResetLink" @click="resetPassword()">Glemt Kodeord?</a>
              <div class="logonUserBtnContainer">
                <button class="btn btnLogonUser" @click="loginEmailPassword">
                  Log ind
                </button>
                <button class="btn redirectCreateUser" @click="menuScreenToggle(4)">
                  Opret Bruger
                </button>
              </div>
            </div>
            <div class="logonCreateUser" v-if="screenOpenStatus == 4">
              <button class="btn returnBtn" @click="menuScreenToggle(1)">
                X
              </button>
              <span class="inputBox inputCreateLogon">
                <p class="p">Brugernavn</p>
                <input class="textCreateName" v-model="username" />
              </span>
              <span class="inputBox inputNameEmail">
                <p class="p">Email</p>
                <input class="textCreateEmail" type="email" v-model="email" />
              </span>
              <span class="inputBox inputPasswordCreate">
                <p class="p">Password</p>
                <input class="textCreatePassword" type="password" v-model="password" />
              </span>
              <span class="inputBox inputClassCreate">
                <p class="p">Klasse</p>
                <v-select class="dropdown" v-model="_class" :options="_classes" />
              </span>
              <span class="inputBox inputTeamCreate">
                <p class="p">Hold - Valgfri</p>
                <v-select class="dropdown" v-model="team" :options="_teams" />
              </span>
              <span class="createUserBtnContainer">
                <button class="btn btnSaveNewUser" @click="createAccountEmailPassword">
                  Tilføj
                </button>
              </span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>

</style>