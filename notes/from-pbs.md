
## bloggins

### simple clean architecture with entities

- simple, "no" deps
- much less likely to run into cycle import while agiling at small scale
- small -> entity pkg, larger -> entity/foo(etc) pkgs

### encapsulated environmental config

### golang binary versioning

### mysteries of middlewarze

### simple clean architecture with service layers

- clientsvc is too simple so photosvc!

### simple random id's

### makeing room for worker routines

### interfaces and p-p-pointers

- (method SendObject has pointer receiver) how does it know?

### slice mutation!!!

### BBD unit

- unit testing is like, .. yeah
- bbd style, pretty common across langs
  - living, runnable documentation
- table driven, works too :)
- ginkgo/gomega to the rescue
  - matcher lib makes sense to me
- 80/20
  - cover % is not so good anyways, how well is a line tested?
- nice to have at min test per pkg

### working with ginkgo/gomega

- . import or inpkg?
- start with small boiler, expand empire
- JustBeforeEach is handy for method under test

### moq, gomock, or handtooled

- prolly moq v handtooled
- handtooled logger copied into pbs is nice
  - copied from where? lol (how to reuse??)
  - better to add these features "under" moq?

### vim generally

### vim-go fo-sho

### placeholder
