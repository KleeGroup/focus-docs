# Implémenter une liste administrable

## Objectifs

L'objectif de cet exercice est de pouvoir afficher une liste potentiellement très longue.
Il sera possible de :

-   Editer ligne par ligne dans une popin qui apparaîtra depuis la droite de l'écran,
-   Rafraichir les éléments de la liste en sortie d'édition d'un élément,
-   Mettre en place un filtre afin d'affiner les éléments affichés dans la liste.

L'API utilisée afin de communiquer avec le serveur est calée sur celle du moteur de recherche.

Normalement vous devriez avoir une page qui ressemble à ça à la fin:

-   Le mode liste :

![](images/admin-country.png)

-   Le mode édition d'une ligne :

![](images/admin-country-edit.png)

## Concepts focus manipulés

### Le ListStore

Focus propose un store de liste accessible dans [focus-core/store/list](https://github.com/KleeGroup/focus-core/blob/master/src/store/list/index.js).
Le store de liste aura les noeuds suivants :

-   `criteria` : Le critère éventuel de recherche qui doit être un objet structuré,
-   `sortBy` : Par quel élement la liste est triée,
-   `sortAsc` : Le caractère ascendant ou non pour le tri,
-   `dataList` : Les élements de la liste (une partie au moins s'il y a pagination),
-   `totalCount` : Le nombre total d'éléments de la liste (utile pour réaliser la pagination).

<!-- prettier-ignore-start -->
!!! warning
    Attention il est nécessaire de fournir au moment de l'instanciation de ce store une clé unique qui permettra de distinguer un store de liste d'un autre, 
    tous les stores ayant les même noeuds
<!-- prettier-ignore-end -->

### Le ListActionBuilder

Le builder permet de créer deux choses :

-   Une `action` qui sera appellée par le composant de liste a chaque fois qu'un changement est opéré dans le store de liste
-   Une fonction qui permet de dispatcher de nouveaux élements dans le store

### Les Composants graphiques

-   Le composant qui est responsable de gérer l'affichage de la liste [focus-components/page/list](https://github.com/KleeGroup/focus-components/tree/master/src/page/list)
-   le composant d'affichage d'une liste dans focus qui est [focus-components/list/selection](https://github.com/KleeGroup/focus-components/tree/master/src/list/selection)

Il est important de comprendre à ce niveau que parmi tous les composants que vous allez créer, seul le composant principal de liste effectuera des requêtes à l'API afin de récupérer les données. Les autres composants auront juste pour tâche de mettre à jour le store de liste en fonction des cas d'usages.

## Création de la page

Commençons par créer un dossier pour l'ensemble des composants de cette page de liste, dans notre dossier de vues: `views`.

La manière dont vous organisez votre dossier de vues vous incombe, cependant une bonne pratique est de regrouper les vues par module puis par écran.

Dans notre cas, nous créons le dossier : `views/movies/movie-list`.

Créons notre composant parent :

```js
// app/views/movies/movie-list/index.jsx

// Libs
import React from "react";

export const MovieListPage = React.createClass({
    render() {
        return (
            <>
                <div>Barre de recherche de films</div>
                <div>Liste de films</div>
                <div>Popin de preview d'un film</div>
            </>
        );
    }
});
```

Notre page de détail est pour l'instant très simple, elle est affichée lorsque l'on navigue vers l'URI `/#movies`.

Il faut donc créer un nouveau fichier pour les routes de movies :

```diff
// app/routes/movie-routes.jsx

// Libs
import React from "react";

// Components
+++ import { MovieListPage } from "../views/movies/movie-list";

export const movieRoutes = [
+++ {
+++     path: "movies",
+++     component: ({ params }) => <MovieListPage />
+++ },
    // [...]
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

## Les pré-requis pour la liste

### Création du store

Dans le dossier `app/stores` de votre application créer un fichier `movie-list.js`.

```js
// app/stores/movie-list.js

// Libs
import ListStore from "focus-core/store/list";

// On créer une instance de ce store avec un identifiant unique
export const movieListStore = new ListStore({ identifier: "movieList" });
movieListStore.name = "MovieListStore";
```

<!-- prettier-ignore-start -->
!!! note
    Nous lui donnons un nom afin d'aider pour les messages de debug.
    Ce n'est pas obligatoire mais c'est mieux.
<!-- prettier-ignore-end -->

### Création du service

Dans le fichier `app/services/movies.js` on va ajouter une méthode pour rechercher des films :

```js
// app/services/movies.js

// Apis
import moviesApi from "../config/server/generated/movies";

export const movieServices = {
    // [...]
    searchMovies({ urlData, data }) {
        return movieApiDriver.searchMovies(urlData, data);
    }
};
```

Un contrat à respecter est obligatoire pour cette API.

-   Les paramètres d'entée de l'API sont de la forme suivante :

```js
const { criteria } = data;
const { skip, top, sortBy, sortAsc } = urlData;
```

-   Les paramètres de sortie de l'API sont de la forme suivante :

```js
{
  dataList: [{id: 1, ...}, ...],
  totalCount: XXX
}
```

Si ce n'est pas le cas et que vous avez directement les data, n'oubliez pas que le type de retour de fetch est une promesse, et qui est donc par définition : **chainable**.
Vous pouvez faire la chose suivante dans votre service pour normaliser les données au format attendu par focus :

<!-- prettier-ignore-start -->
```js
return movieApiDriver
    .searchMovie(urlData, data)
    .then(data => ({ 
        dataList: data, 
        totalCount: data.length 
    }));
```
<!-- prettier-ignore-end -->

Mais il est préférable que votre serveur vous retourne un objet déjà bien construit.

### Création de l'action

Créer un fichier pour les actions liées à votre entité

```js
// app/actions/movies.js

// Libs
import listActionBuilder from "focus-core/list/action-builder";

// Stores
import { movieListStore } from "../stores/movie-list";

// Services
import { movieServices } from "../services/movies";

export const movieActions = {
    // [...]
    searchMovies = listActionBuilder({
        service: movieServices.searchMovies,
        identifier: "movieList",
        getListOptions: () => movieListStore.getValue()
    });
};
```

Le builder `listActionBuilder` va retourner un objet avec deux fonction :

-   `load` : La fonction de load de la liste,
-   `updateProperties` : La fonction pour mettre à jour les élements du store.

On a maintentant tout ce qu'il nous faut afin de créer le composant.

## Les composants de l'écran

### Création du composant de liste

Commençons par créer le composant de liste.

```js
// app/views/movies/movie-list/movie-list.jsx

// Libs
import React, { PropTypes } from "react";

// Stores
import { movieListStore } from "../../../stores/movie-list";

// Actions
import { movieActions } from "../../../actions/movies";

// Components
import { component as List } from "focus-components/page/list";
import { MovieLine } from "./movie-line";

const columns = {
    title: { label: "movie.title" },
    productionYear: { label: "movie.productionYear" }
};

export const MovieList = ({ onLineClick }) => {
    return (
        <List
            // L'action qui charge la liste
            action={movieActions.searchMovies}
            // Les colonnes à afficher
            columns={columns}
            // Dire à la liste qu'elle n'est pas sélectionnable
            isSelection={false}
            // La ligne à utliser dans la liste
            LineComponent={MovieLine}
            // Le handler de click sur la ligne
            onLineClick={onLineClick}
            // Le store sur lequel le composant doit s'abonner.
            store={movieListStore}
        />
    );
};
MovieList.propTypes = {
    onLineClick: PropTypes.func.isRequired
};
```

### Création du composant de ligne

Nous allons maintenant créer le contenu de la ligne.
Nous avons besoin d'aller chercher la définition de l'entité qui contient l'ensemble des métadonnées associées à chacun de champ de la ligne.

```js
// Libs
import React from "react";

// Components
import { mixin as LineMixin } from "focus-components/list/selection/line";

export const MovieLine = React.createClass({
    displayName: "MovieLine",
    mixins: [LineMixin],
    definitionPath: "movie", // Définition de l'entité
    renderLineContent(data) {
        return (
            <tr onClick={() => this.props.onLineClick(data.id)}>
                <td>{this.textFor("title")}</td>
                <td>{this.textFor("productionYear")}</td>
            </tr>
        );
    }
});
```

La ligne est maintenant créée, la liste est fonctionnelle.
On peut ajouter la liste nouvellement créée à la page :

```diff
// app/views/movies/movie-list/index.jsx

// Libs
import React from "react";

+++ // Components
+++ import { MovieList } from "./movie-list";

export const MovieListPage = React.createClass({
    render() {
        return (
            <div>Barre de recherche de films</div>
---         <div>Liste de films</div>
+++         <MovieList onLineClick={() => { /* RAF */ }} />
            <div>Popin de preview d'un film</div>
        );
    }
});
```

### Création du filtre de recherche

On va créer un filtre de recherche. La logique de ce composant est d'avoir un texte à saisir et une action à appeller lorsque ce texte est modifié.
Le composant va donc prendre en entrée une props : `onFilterChange`.

```js
// app/views/movies/movie-list/movie-criteria.jsx

// Libs
import React, { PropTypes } from "react";
import { debounce } from "lodash";
import { translate } from "focus-core/translation";

export const MovieCriteria = React.CreateClass({
    displayName: "MovieCriteria",
    mixins: [formMixin],
    definitionPath: "movie",
    getInitalState() {
        return { title: "" };
    },
    render() {
        const { filter, onFilterChange } = this.props;
        return (
            <div data-demo="movie-criteria">
                {this.fieldfor("title", {
                    value: filter,
                    onChange: title => onFilterChange(title)
                })}
            </div>
        );
    }
});
MovieCriteria.propTypes = {
    filter: PropType.string,
    onFilterChange: PropTypes.func.isRequired
};
```

Nous devons maintenant ajouter ce composant dans le composant principal.
Et ajouter une fonction qui permettra de dispatcher le query de recherche.

Ajout dans le render :

```diff
// app/views/movies/movie-list/index.jsx

// Libs
import React from "react";

+++ // Store
+++ import { movieListStore } from "../../../stores/movie-list";

+++ // Actions
+++ import { movieActions } from "../../../actions/movies";

// Components
import { MovieList } from "./movie-list";
import { MovieCriteria } from "./movie-criteria";

+++ function onFilterChange(title) {
+++     moviesActions.searchMovies.updateProperties({
+++         criteria: { title }
+++     });
+++ }

export const MovieListPage = React.createClass({
    render() {
+++     const properties = movieListStore.getValue();
        return (
---         <div>Barre de recherche de films</div>
+++         <MovieCriteria
+++             filter={properties.criteria.title}
+++             onFilterChange={onFilterChange}
+++         />
            <MovieList onLineClick={() => { /* RAF */ }} />
            <div>Popin de preview d'un film</div>
        );
    }
});
```

<!-- prettier-ignore-start -->
!!! note
    Utiliser la fonction `updateProperties` déclanche automatiquement une recherche
<!-- prettier-ignore-end -->

### Création de la popin d'édition

Ici pour simplifier nous ne mettrons pas le formulaire car la logique est strictement identique à celle de création de l'écran de détail. Voir [page de détail](./detail-page.md)

```js
// app/views/movies/movie-list/popin.jsx

// Libs
import React, { PropTypes } from "react";
import { translate } from "focus-core/translation";

// Components
import Panel from "focus-components/components/panel";
import { component as Popin } from "focus-components/components/popin";

export function MoviePopin({ id, isOpen, onPopinClose }) {
    return (
        <Popin open={isOpen} onPopinClose={onPopinClose}>
            <h4>{translate("movie.popin.title")}</h4>
            <div>{`Popin d'édition pour le film ${id}`}</div>

            {/* Affichage du formulaire de détail */}
        </Popin>
    );
}
MoviePopin.propTypes = {
    id: PropTypes.number,
    isOpen: PropTypes.boolean.isRequired,
    onPopinClose: PropTypes.func.isRequired
};
```

Cette poin doit maintenant être ajoutée dans le composant principal de la liste .

```diff
// app/views/movies/movie-list/index.jsx

// Libs
import React from "react";

// Actions
import { movieActions } from "../../../actions/movies";

// Components
import { MovieList } from "./movie-list";
import { MovieCriteria } from "./movie-criteria";
+++ import { MoviePopin } from "./movie-popin";

function onFilterChange(title) {
    moviesActions.searchMovies.updateProperties({
        criteria: {
            title
        }
    });
}

export const MovieListPage = React.createClass({
+++ getInitialState() {
+++     return { id: undefined };
+++ },
+++ onPopinClose() {
+++     this.setState({ id: undefined });
+++ },
+++ openPopin(id) {
+++     this.setState({ id });
+++ },
    render() {
        return (
            <MovieCriteria onFilterChange={onFilterChange} />
---         <MovieList onLineClick={() => { /* RAF */ }} />
+++         <MovieList onLineClick={id => this.openPopin(id)} />
---         <div>Popin de preview d'un film</div>
+++         <MoviePopin
+++             id={this.state.id}
+++             isOpen={this.state.id !== undefined}
+++             onPopinClose={() => this.onPopinClose()}
+++         />
        );
    }
});
```

Nous avons fait les modifications suivante :

-   Ajout d'un state pour savoir l'identifiant du movie sur lequel on a cliqué
-   Donné la méthode `openPopin` au composant `MovieList` pour ouvrir la popin
-   Donné la méthode `onPopinClose` au composant `MoviePopin` pour fermer la popin

Ici, le **trick** est d'utiliser la propriété `id` pour savoir si la popin est ouverte ou fermée.

## Epilogue

![](https://media.giphy.com/media/lgIyvBoSKEhuo/giphy.gif)
