---
title: Connecting Components with Reactive Forms
description: >-
  How to use RxJS and Reactive Forms to connect presentational and container
  components
date: "2017-08-04T06:22:00.299Z"
categories: []
keywords: []
cardImage: "./title.jpeg"
slug: /@cartant/connecting-components-with-reactive-forms-55f56fce2aad
---

![Blurred lights](title.jpeg "Photo by Sebastian Muller on Unsplash")

In his article [_Presentational and Container Components_](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0), Dan Abramov discusses separating presentational and container components in React. The idea is general and separating Angular components in a similar manner is an approach that some developers choose to take.

With Angular, the separation of container and presentational components gets interesting when [Reactive Forms](https://angular.io/guide/reactive-forms) are added to the mix. A reactive form introduces an observable, subscriptions to which receive notifications of the form’s — or one of its controls’ — changed values. The notifications make it easy to implement features like debounced searches using RxJS. However, if such features are implemented in presentational components, they will no longer be purely presentational.

How can Reactive Forms be used without compromising the simplicity of presentational components? Let’s look at a solution that uses RxJS.

Usually, presentational components communicate with container components using an `Output` property that’s an `EventEmitter`. However, when using Reactive Forms, that would involve converting the notifications received from the form’s `valueChanges` observable to events. And, in the container component, to be used with an observable service, the events would need to be converted back into an observable. That seems complicated.

Instead, the presentational component could accept an `Input` observer that could be subscribed to the `valueChanges` observable. In fact, it can accept a [`PartialObserver`](https://github.com/ReactiveX/rxjs/blob/5.4.2/src/Observer.ts#L22), as there is no particular requirement for the observer to implement `error` or `complete`.

A presentational component that displays a list of GitHub users with usernames that start with a typed-in value would look something like this:

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  selector: "user-search",
  template: `
    <form [formGroup]="form">
      <md-input-container>
        <input formControlName="username" mdInput placeholder="Username" />
      </md-input-container>
    </form>
    <md-list>
      <md-list-item *ngFor="let user of users">
        <p md-line>{{ user.login }}</p>
        <p md-line>{{ user.public_repos }} public repo(s)</p>
      </md-list-item>
    </md-list>
  `
})
export class UserSearchComponent implements OnDestroy, OnInit {
  @Input() public observer: PartialObserver<string>;
  @Input() public users: User[];

  public form: FormGroup;
  private _subscription: Subscription;

  constructor(formBuilder: FormBuilder) {
    this.form = formBuilder.group({ username: [] });
  }

  ngOnDestroy(): void {
    this._subscription.unsubscribe();
  }

  ngOnInit(): void {
    this._subscription = this.form.valueChanges
      .map(values => values.username)
      .subscribe(this.observer);
  }
}
```

The component accepts an observer and an array of users — the results of the search. It subscribes the observer to the form’s `valueChanges`, mapping the observable’s form values to the typed-in username. And it displays the users in a list.

The container component would look something like this:

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  selector: "user-search-container",
  template: `
    <user-search [observer]="observer" [users]="users | async"> </user-search>
  `
})
export class UserSearchContainer {
  public observer: PartialObserver<string>;
  public users: Observable<User[]>;
  private _subject: Subject<string>;

  constructor(service: UserSearchService) {
    this._subject = new Subject<string>();
    this.observer = this._subject;
    this.users = this._subject
      .debounceTime(1000)
      .distinctUntilChanged()
      .switchMap(username => this.service.searchUsers(username));
  }
}
```

The container component passes a `Subject` to the presentational component — to receive the notifications for the changes to the typed-in username. Subjects are both observers and observables, so the `users` observable can be composed directly from the subject.

The service injected into the container component would implement a `searchUsers` method, something like this:

```ts
@Injectable()
export class UserSearchService {
  constructor(private _http: Http) {}

  searchUsers(username: string): Observable<User[]> {
    return username
      ? this._http
          .get(`https://api.github.com/search/users?q=${username}`)
          .map(response => response.json())
          .map(content => content.items)
      : Observable.of<User[]>([]);
  }
}
```

With this arrangement, the typed-in values will flow from the presentational component’s reactive form, to the container component’s subject, where they will be debounced and then passed to the search service. The users returned from the service will then flow back to the presentational component, where they will be received as an array of `User` entities due to the `async` pipe used in the container component.

Using subjects to connect observables in this manner is something that happens a lot in RxJS. In fact, it’s how some operators — like [`share`](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-share) — are implemented.

If you are interested in reading more about subjects and their uses, Ben Lesh’s [_On the Subject of Subjects (in RxJS)_](https://medium.com/@benlesh/on-the-subject-of-subjects-in-rxjs-2b08b7198b93) is a good place to start.
