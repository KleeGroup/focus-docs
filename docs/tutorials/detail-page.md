# Implémenter une page de détail

## Objectifs

L'objectif de cet exercise et de pouvoir afficher une page de détail d'un film.

Elle est constituée :

-   d'un header ([Cartridge](https://github.com/KleeGroup/focus-components/blob/master/src/page/mixin/cartridge-behaviour.js))
-   d'une zone de navigation rapide ([Scrollspy](https://github.com/KleeGroup/focus-components/tree/master/src/components/scrollspy-container))
-   d'un ou plusieurs blocks de détail ([Panel](https://github.com/KleeGroup/focus-components/tree/master/src/components/panel))

Une page de détail ressemble à ça :

![page de détail focus](images/detail.png)

## Concepts focus manipulés

### Le CoreStore

Focus propose un store simple [CoreStore](https://github.com/KleeGroup/focus-core/blob/master/src/store/CoreStore.js) qui permet de stocker n-noeuds par store.
L'idée est que chaque store couvre un thème fonctionnel de l'application.
Par exemple, un store `movieStore` qui aura les noeuds : `resume`, `characteristics`, `actors`.

Chaque noeud aura plusieurs propriétés associés : `value`, `status`, `error`.

<!-- prettier-ignore-start -->
!!! warning
    Le CoreStore pour savoir s'il accepte la payload d'une action se base uniquement sur le nom du noeud. Si plusieurs stores ont un noeud avec
    un nom identique, alors il n'y a pas moyen de les différencier.
<!-- prettier-ignore-end -->

### Le ActionBuilder

Le [ActionBuilder](https://github.com/KleeGroup/focus-core/blob/master/src/application/action-builder.js) va permettre de créer simplement une
action qui va échanger avec le server pour charger ou sauvegarder des données.

Lors de l'exécution d'une action, les opérations suivantes sont effectuées :

1. Modifie le statut du noeud avec la valeur de `preLoading` (par défaut: `loading`) et passe `isLoading` à vrai
2. Si `shouldDumpStoreOnActionCall` est à vrai, alors vide la valeur du noeud
3. Appel le `service` configuré et lui donne la `payload`
4. Si on a configuré un `postService`, on lui donne le retour du `service` pour transformation
5. Dispatch le resultat du serveur dans le store, met le statut du noeud à `status` et `isLoading` à faux

En cas de retour HTTP avec un code d'erreur, on dispatch dans le champ `error` du noeud

Exemple de configuration d'un builder :

```js
const action: payload => Promise<void> = actionBuilder({
    // Noeud ciblé
    node: "movieCharacteristics",
    // Statut pendant le chargement/sauvegarde
    preLoading: "loading",
    // Service à appeler
    service: movieServices.loadMovieCharacteristics,
    // Si on vide le noeud pendant le chargement
    shouldDumpStoreOnActionCall: true,
    // Transformation du retour server
    postService: identity,
    // Le statut du noeud à la fin de l'appel
    status: "loaded"
});
```

### Le FormMixin

// TODO - décrire le formMixin

## Création de la page

Commençons par créer un dossier pour l'ensemble des composants de cette page de détail, dans notre dossier de vues: `views`.

La manière dont vous organisez votre dossier de vues vous incombe, cependant une bonne pratique est de regrouper les vues par module puis par écran.

Dans notre cas, nous créons le dossier : `views/movies/movie-detail`.

Créons notre composant parent :

```js
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from "react";

export const MovieDetailPage = React.createClass({
    render() {
        const { id } = this.props;

        return <div>{`Page de détail du movie ${id}`}</div>;
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

Notre page de détail est pour l'instant très simple, elle n'a qu'une _prop_, `id`, qui lui sera fournie par le routeur lorsque l'on navigue vers l'URI `/#movies/{id}`.

Il faut donc créer un nouveau fichier pour les routes de movies :

```js
// app/routes/movie-routes.jsx

// Libs
import React from "react";

// Components
import { MovieDetailPage } from "../views/movies/movie-detail";

export const movieRoutes = [
    {
        path: "movies/:id",
        component: ({ params }) => <MovieDetailPage id={params.id} />
    }
];
```

Puis l'enregistrer auprès du router :

```diff
// app/routes/index.js

// Components
import AppLayout from "../components/app-layout";

// Routes
import { homeRoutes } from "./home-routes";
+++ import { movieRoutes } from "./movie-routes";

export default {
    path: `${__BASE_URL__}`,
    component: AppLayout,
    indexRoute: { onEnter: ({ params }, replace) => replace(`${__BASE_URL__}home`) },
---    childRoutes: [...homeRoutes]
+++    childRoutes: [...homeRoutes, ...movieRoutes]
};
```

## Les pré-requis pour l'écran

### Création du store

Dans le dossier `app/stores` de votre application créer un fichier `movie-detail.js`.

```js
// app/stores/movie-detail.js

// Libs
import { CoreStore } from "focus-core/store";

// On créer une instance de ce store
export const movieDetailStore = new CoreStore({
    movieCharacteristics: "movieCharacteristics"
});
movieDetailStore.name = "movieDetailStore";
```

<!-- prettier-ignore-start -->
!!! note
    Nous lui donnons un nom afin d'aider pour les messages de debug.
    Ce n'est pas obligatoire mais c'est mieux.
<!-- prettier-ignore-end -->

### Création des APIs

Notre entité **movie** dispose de deux APIs de manipulation, un **load** et et un **save**.

Il est necessaire de créer le fichier `config/server/movies.js` qui contient les URIs exposées par l'API.
Le mieux pour cela est d'utiliser le paquet [focus-service-generator](https://github.com/KleeGroup/focus-service-generator) (il est configuré avec le starter kit)

Exécutez :

```bash
npm run generate
```

Cela va générer un fichier de la forme suivante :

```js
// app/config/server/generated/movies.js

import apiDriverBuilder from "../../../utilities/api-driver";

// Movies
export default apiDriverBuilder({
    getMovie: {
        url: __API_ROOT__ + "/api/movie/${id}",
        method: "GET"
    },
    updateMovie: {
        url: __API_ROOT__ + "/api/movie/${id}",
        method: "PUT"
    }
    // [...]
});
```

Appelons le load au chargement de la page de détail afin de disposer des données dans le store et donc dans les vues :

<!-- prettier-ignore-start -->
!!! note
    La signature des méthodes API sont de la forme : `(uriParam, bodyParam, misc)`
<!-- prettier-ignore-end -->

### Création des services

Dans le fichier `app/services/movies.js` on va ajouter une méthode pour charger et sauvegarder un film :

```js
// app/services/movies.js

// Apis
import moviesApi from "../config/server/generated/movies";

export const movieServices = {
    loadMovieCharacteristics(id) {
        return moviesApi.getMovie({ id });
    },
    updateMovieCharacteristics(movie) {
        const { id } = movie;
        return moviesApi.updateMovie({ id }, movie);
    }
};
```

### Création des actions

Créer un fichier pour les actions liées à votre entité

```js
// app/actions/movies.js

// Libs
import actionBuilder from "focus-core/application/action-builder";

// Services
import { movieServices } from "../services/movies";

export const movieActions = {
    movieCharacteristics: {
        load: actionBuilder({
            node: "movieCharacteristics",
            service: movieServices.loadMovieCharacteristics,
            shouldDumpStoreOnActionCall: true,
            status: "loaded"
        }),
        save: actionBuilder({
            node: "movieCharacteristics",
            service: movieServices.updateMovieCharacteristics,
            shouldDumpStoreOnActionCall: false,
            status: "saved"
        })
    }
};
```

## Les composants de l'écran

### Premier panel de détail

Nous allons créer un premier panel, pour afficher et éditer les caractéristiques principales de notre **movie**.

```js
// app/views/movies/movie-detail/movie-characteristics.jsx

// Libs
import React, { PropTypes } from "react";

// Components
import { Panel } from "focus-components/components";

export const MovieCharacteristics = React.createClass({
    render() {
        const { id } = this.props;
        return (
            <Panel title="view.movie.detail.characteristics">
                <div>{`panel de caractéristiques du film ${id}`}</div>
            </Panel>
        );
    }
});
MovieCharacteristics.propTypes = {
    id: PropTypes.number.isRequired
};
```

On instancie le composant dans la page principale :

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from "react";

+++ // Components
+++ import { MovieCharacteristics } from "./movie-characteristics";

export const MovieDetailPage = React.createClass({
    render() {
        const { id } = this.props;

---        return <div>{`Page de détail du movie ${id}`}</div>;
+++        return <MovieCharacteristics id={id} />;
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

Notre premier panel est maintenant prêt à recevoir les données depuis le store applicatif, et à faire les appels serveurs permettant le chargement de l'entité dans les store.

### Chargement de l'entité

Le chargement de l'entité passe par les stores applicatif, en suivant l'architecture **flux**.

Il est donc nécessaire de lancer l'action qui va effectuer le chargement serveur de l'entité, et de s'abonner aux changements du store de l'entité afin de bénéficier des données retournées par le serveur.

Appelons le load au chargement de la page de détail afin de disposer des données dans le store et donc dans les vues :

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from 'react';

+++ // Actions
+++ import { movieActions } from '../../../actions/movies';

export const MovieDetailPage = React.createClass({
+++     componentDidMount() {
+++         movieActions.movieCharacteristics.load();
+++     },

    render() {
        const { id } = this.props;

        return <MovieCharacteristics id={id} />;
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

### Affichage de l'entité

Maintenant que l'entité est chargée en mémoire au montage de la page de détail, il est possible de l'afficher dans le panel `MovieCharacteristics`.

Pour cela, nous allons utiliser le **formMixin** dans le panel. Il va nous faire bénéficier de l'ensemble des fonctionnalités du form focus, à savoir :

-   l'abonnement aux stores,
-   le chargement des définitions avec domaines associés,
-   les helpers de field,
-   et le branchement aux actions de l'entité `load` et `save`.

```diff
// app/views/movies/movie-detail/movie-characteristics.jsx

// Libs
import React, { PropTypes } from 'react';
+++ import { mixin as formMixin } from 'focus-components/mixin/form';

// Actions & Stores
import { movieActions } from '../../../actions/movies';
+++ import { movieDetailStore } from '../../../stores/movies';

// Components
import { Panel } from 'focus-components/components';

export const MovieCharacteristics = React.createClass({
+++ // Ajout du form mixin
+++ mixins: [formMixin],
+++ // Définition de notre entité
+++ definitionPath: 'movie',
+++ // Abonnement au store
+++ stores: [{ store: movieDetailStore, properties: ['movieCharacteristics'] }],
+++ // Donne les actions au form
+++ action: movieActions.movieCharacteristics,
--- render() {
---     const { id } = this.props;
+++ // render() est déjà défini par le formMixin
+++ renderContent() {
        return (
---         <Panel title='view.movie.detail.characteristics'>
+++         <Panel
+++             actions={this._renderActions}
+++             title='views.movie.detail.characteristics'
+++         >
---         <div>{`panel de caractéristiques du film ${id}`}</div>
+++             {/* Les fieldFor sont des fonctions helper */}
+++             {/* pour afficher et éditer un champ avec label} */}
+++             {this.fieldFor('title')}
+++             {this.fieldFor('originalTitle')}
+++             {this.fieldFor('keywords')}
+++             {this.fieldFor('runtime')}
+++             {this.fieldFor('movieType')}
+++             {this.fieldFor('productionYear')}
            </Panel>
        );
    }
});
MovieCharacteristics.propTypes = {
    id: PropTypes.number.isRequired
};
```

Il est important d'ajouter une **prop** depuis le parent vers `MovieCharacteristics` afin de lui signaler de ne pas effectuer le **load** à son chargement, le composant parent le fait déjà dans son `componentDidMount`. Par défaut, le **formMixin** se chargerait de le faire en appelant : `this.action.load(this.props.id)`.

Faire le chargement dans le parent permet de s'assurer qu'il n'est fait qu'une fois pour l'ensemble des enfants.

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from 'react';

// Actions
import { movieActions } from '../../../actions/movies';

// Components
import { MovieCharacteristics } from "./movie-characterics";

export const MovieDetailPage = React.createClass({
    componentDidMount() {
        movieActions.movieCharacteristics.load();
    },

    render() {
        const { id } = this.props;

---        return <MovieCharacteristics id={id} />;
+++        return <MovieCharacteristics id={id} hasLoad={false} />;
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

### Ajout second panel

Nous allons ajouter un panel propre au synopsis du movie, afin de séparer l'information et de rendre plus limpide la lecture de la page de détail.

Le synopsis est un attribut de notre entité, on bénéficie donc déjà de l'information dans le noeud `movieCharacteristics` du `movieDetailStore`.

```js
// app/views/movies/movie-detail/movie-synopsis.jsx

// Libs
import React, { PropTypes } from "react";
import { mixin as formMixin } from "focus-components/mixin/form";

// Actions & Stores
import { movieActions } from "../../../actions/movies";
import { movieDetailStore } from "../../../stores/movies";

// Components
import { Panel } from "focus-components/components";

export const MovieSynopsis = React.createClass({
    mixins: [formMixin],
    definitionPath: "movie",
    stores: [{ store: movieDetailStore, properties: ["movieCharacteristics"] }],
    action: movieActions.movieCharacteristics,

    renderContent() {
        return (
            <Panel actions={this._renderActions} title="views.movie.detail.synopsis">
                {this.fieldFor("synopsis")}
            </Panel>
        );
    }
});
MovieSynopsis.propTypes = {
    id: PropTypes.number.isRequired
};
```

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from "react";

// Actions
import { movieActions } from "../../../actions/movies";

// Components
import { MovieCharacteristics } from "./movie-characterics";
+++ import { MovieSynopsis } from "./movie-synopsis";

export const MovieDetailPage = React.createClass({
    componentDidMount() {
        movieActions.movieCharacteristics.load();
    },

    render() {
        const { id } = this.props;

        return (
            <>
                <MovieCharacteristics id={id} hasLoad={false} />;
+++             <MovieSynopsis id={id} hasLoad={false} />;
            </>
        );
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

### Ajout navigation rapide

La navigation rapide est rapidemment mise en place. En effet, il s'agit d'un simple wrapper [ScrollspyContainer](https://github.com/KleeGroup/focus-components/blob/master/src/components/scrollspy-container/index.js) autour de nos panels, qui va de lui-même lister les panels et mettre en place la navigation.

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from "react";

// Actions
import { movieActions } from "../../../actions/movies";

// Components
+++ import ScrollspyContainer from "focus-components/components/scrollspy-container";
import { MovieCharacteristics } from "./movie-characterics";
import { MovieSynopsis } from "./movie-synopsis";

export const MovieDetailPage = React.createClass({
    componentDidMount() {
        movieActions.movieCharacteristics.load();
    },

    render() {
        const { id } = this.props;

        return (
---         <>
+++         <ScrollspyContainer>
                <MovieCharacteristics id={id} hasLoad={false} />;
                <MovieSynopsis id={id} hasLoad={false} />;
---         </>
+++         </ScrollspyContainer>
        );
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

### Ajout du header

Pour finaliser notre page de détail, nous allons ajouter des éléments dans le header (ou Cartridge).

Le header a deux modes de fonctionnement : déplié et replié. Il faut donc lui fournir un composant pour chaque mode.

Le store étant déjà peuplé par le composant de la page de détail, on peut donc simplement utiliser le **formMixin** pour s'abonner au store et afficher les champs qui nous intéressent.

```js
// app/views/movies/movie-detail/header-expanded.jsx

// Libs
import React from "react";
import { mixin as formMixin } from "focus-components/mixin/form";

// Actions & Stores
import { movieDetailStore } from "../../../stores/movies";

// Components
import Poster from "../components/poster";

export const MovieHeaderExpanded = React.createClass({
    displayName: "MovieHeaderExpanded",
    mixins: [formMixin],
    definitionPath: "movie",
    stores: [{ store: movieDetailStore, properties: ["movieCharacteristics"] }],
    renderContent() {
        const { poster, title } = this.state;

        return (
            <div>
                {poster && <Poster poster={poster} title={title} />}
                <h3>{this.textFor("title")}</h3>
                <h5>{this.textFor("movieType")}</h5>
                <h6>{this.textFor("productionYear")}</h6>
                <div>{this.textFor("shortSynopsis")}</div>
            </div>
        );
    }
});
```

Le composant de header replié est similaire, avec quelques informations en moins pour occuper moins de place.

```js
// app/views/movies/movie-detail/header-collapsed.jsx

// Libs
import React from "react";
import { mixin as formMixin } from "focus-components/mixin/form";

// Actions & Stores
import { movieDetailStore } from "../../../stores/movies";

export const MovieHeaderCollapsed = React.createClass({
    displayName: "MovieHeaderCollapsed",
    mixins: [formMixin],
    definitionPath: "movie",
    stores: [{ store: movieDetailStore, properties: ["movieCharacteristics"] }],
    renderContent() {
        return (
            <div>
                <h3>{this.textFor("title")}</h3>
                <h6>{this.textFor("productionYear")}</h6>
            </div>
        );
    }
});
```

Nos composants de header sont définis, mais ne sont pas injectés dans le header.

Pour ce faire, on peut utiliser le [cartridgeBehaviour](https://github.com/KleeGroup/focus-components/blob/master/src/page/mixin/cartridge-behaviour.js), qui va permettre d'indiquer à la page quels composants afficher dans le header, ainsi que les actions globales disponibles en tant que boutons dans le header.

Modifions la page parente afin de lui donner le cartridgeBehaviour et d'envoyer les composants dans le header.

```diff
// app/views/movies/movie-detail/index.jsx

// Libs
import React, { PropTypes } from "react";
+++ import { cartridgeBehaviour } from "focus-components/page/mixin";

// Actions
import { movieActions } from "../../../actions/movies";

// Components
import ScrollspyContainer from "focus-components/components/scrollspy-container";
+++ import BackButton from "focus-components/components/button-back/";
import { MovieCharacteristics } from "./movie-characterics";
import { MovieSynopsis } from "./movie-synopsis";
+++ import { MovieHeaderCollapsed } from "./header-collapsed";
+++ import { MovieHeaderExpanded } from "./header-expanded";

export const MovieDetailPage = React.createClass({
    componentDidMount() {
        movieActions.movieCharacteristics.load();
    },
+++ mixins: [cartridgeBehaviour],
+++ cartridgeConfiguration() {
+++     const props = { hasLoad: false, hasForm: false }; // props qui seront données aux composants du header
+++     return {
+++         barLeft: { component: BackButton }, // On ajoute le bouton Back en haut à gauche de la page
+++         cartridge: { component: MovieHeaderExpanded, props },
+++         summary: { component: MovieHeaderCollapsed, props },
+++         actions: {
+++             primary: [
+++                 {
+++                     // action d'exemple
+++                     label: "Imprimer",
+++                     icon: "print",
+++                     action: () => {
+++                         alert("todo print");
+++                     }
+++                 }
+++             ],
+++             secondary: []
+++         }
+++     };
+++ }
    render() {
        const { id } = this.props;

        return (
            <ScrollspyContainer>
                <MovieCharacteristics id={id} hasLoad={false} />;
                <MovieSynopsis id={id} hasLoad={false} />;
            </ScrollspyContainer>
        );
    }
});
MovieDetailPage.propTypes = {
    id: PropTypes.number.isRequired
};
```

## Epilogue

![](https://media.giphy.com/media/lgIyvBoSKEhuo/giphy.gif)
