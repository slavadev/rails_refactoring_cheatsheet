# Rails Refactoring Cheatsheet
Based on [How To Refactor Big Rails Projects](https://medium.com/@korolvs/how-to-refactor-big-rails-projects-12fc4e4ddcd2)

## Table of Content
- [General](#general)
- [Model](#model)
- [Repository](#repository)
- [Operation](#operation)
- [Controller](#controller)
- [View](#view)

## General
- Add comments and documentation using [YARD](https://yardoc.org/)
  ```ruby
  # Creates a dog and booth for a dog
  #
  # @param context [Hash] context of operation(current user, current city etc.)
  # @param params [Hash, ApplicationController::Params] attributes of new dog
  #
  # @return [Dog]
  #
  # @raise [UnauthorizedError] if action is forbidden
  # @raise [ValidationError] if dog or booth is not valid
  #
  # @see DogsRepository
  # @see BoothsRepository
  def call(context:, params:)
    #...
  end
  ```
- Try to write code that will be easy to test(create small methods that you can easy stub)
  ```ruby
  # In main code
  def repo
    DogsRepository.new
  end

  def call(params)
    dog = repo.create(params)
    #...
  end

  # In specs
  let(:repo) { instance_double('DogsRepository') }
  before do
    allow(create_operation).to receive(:repo).and_return(repo)
    allow(repo).to receive(:create).and_return(dog)
    # or
    allow(repo).to receive(:create).and_raise(ValidationError)
  end
  ```
- Don't forget to add integration tests if you use stubbing and mocking a lot
- Follow some styleguide. Check your code with [Rubocop](https://github.com/rubocop-hq/rubocop)
- Ask your teammembers if you hesitating at any point

## Model
### Good
- Place in the model only validations, associations, and methods that actually are related to the model.
  ```ruby
  class Dog < ApplicationRecord
    belongs_to :breed
    has_one :booth
    validate :name, presence: true
    validates :age, numericality: { greater_than: 0 }, presence: true

    # Barking
    def bark
      #...
    end

    private

    def am_i_cute?
      true
    end
  end
  ```
- You can keep in the model all "extentions" you use
  ```ruby
  class Dog < ApplicationRecord
    acts_as_paranoid
    #...
    dragonfly_accessor :image
  end
  ```

### Bad
- Don't use callbacks like `after_*` and `before_*`. Put this code in `Repository` if it is DB-related code and in `Operation` if it is business logic
  ```ruby
  class Dog < ApplicationRecord
    before_save :calculate_size # Bad - Move it to Repository
    after_create :create_booth # Bad - Move it to Operation
  end
  ```
- Don't use scopes in the model. Move them to `Repository`
  ```ruby
  class Dog < ApplicationRecord
    scope :all, -> { where('age > 1') } # Bad - Move it to Repository

    # Bad - Move it to Repository
    def self.puppies
      Dog.where('age < 1')
    end
  end
  ```
- Don't use nested attributes for associations. Move creating, updating and deleting associations to `Operation`
  ```ruby
  class Dog < ApplicationRecord
    accepts_nested_attributes_for :booth # Bad - Move this logic to Operation
  end
  ```
- Don't call Repositories and Operations from the model. Repositories should be called from Operations and Operations should be called from other operations or layer above them(Controller, API, Console etc.)
  ```ruby
  class Dog < ApplicationRecord
    def bark()
      similar_dog = DogsRepository.find_similar_to(self) # Bad - Move to Operation
      DogsOperations::Bark.new.call(similar_dog) # Bad - Move to Operation
    end
  end
  ```

## Repository
### Good
- Place here all your code which is responsible for querying(such as scopes) and managing entities(like creating and updating)
  ```ruby
  class DogsRepository
    attr_reader :model
    
    def initialize(model: Dog)
      @model = model
    end
    #...
    def all()
      model.where('age > 1.0') # was a default scope
    end

    #...
    def with_breed(breed)
      all.where(breed: breed)
    end

    #...
    def pugs()
      all.joins(breed: {name: 'Pug'})
    end

    #...
    def puppies()
      Dog.where('age <= 1.0')
    end

    #...
    def find_by_id(id)
      dog = all.find_by(id: id)
      raise NotFoundError, {id: id} unless dog
      dog
    end

    #...
    def create(params)
      dog = model.new(params)
      dog.size = dog.get_size_from_breed_and_age # was a callback before
      raise ValidationError, dog unless dog.save
      dog
    end
  end
  ```
- It's ok to have different repositories for one model
  ```ruby
  class Dogs::SmallDogsRepository
    #...
  end
  
  class Dogs::OldDogsRepository
    #...
  end
  ```
### Bad
- Don't change other models in the repository of the model. It should be done in `Operation` and use another `Repository`
  ```ruby
  class DogsRepository
    #...
    def create(params)
      dog = Dog.create(params)
      booth = Booth.create(dog) # Bad - Move it to operation and use another repository
      #...
    end
  ```
- Don't call other Repositories and Operations from the repository. Repositories should be called from Operations and Operations should be called from other operations or layer above them(Controller, API, Console etc.)
  ```ruby
  class DogsRepository
    #...
    def create(params)
      dog = Dog.create(params)
      booth = BoothsRepository.new.create(dog: dog) # Bad - Move to Operation
      DogsOperations::Bark.new.call(dog) # Bad - Move to Operation
      #...
    end
  ```

## Operation
### Good
- Place here a code related to one business operation. Operation should have one public method `call`.
  ```ruby
  module DogsOperations
    class Create
      attr_reader :dogs_repo, :booths_repo

      def initialize(dogs_repo: DogsRepository.new, booths_repo: BoothsRepository.new)
        @dogs_repo = dogs_repo
        @booths_repo = booths_repo
      end

      #...
      def call(context:, params:)
        authorize(context)
        dog_params = prepare_params(params)
        dog = dogs_repo.create(dog_params)
        create_booth_for(dog)
        dog
      end
      
      #...
    end
  end
  ```
- It's ok to call other operations from `Operation`. Place all dependencies in `initialize` method and create an `attr_reader` for each dependency to make it easy to mock in tests.
  ```ruby
  module DogsOperations
    class Create
      attr_reader :dogs_repo, :bark_operation

      def initialize(dogs_repo: DogsRepository.new, bark_operation: DogsOperations::Bark.new)
        @dogs_repo = dogs_repo
        @bark_operation = bark_operation
      end

      #...
    end
  end
  
  # In tests
  describe DogsOperations::Create do
    let(:repo) { instance_double('DogsRepository') }
    let(:bark) { instance_double('DogsOperations::Bark') }
    let(:create_operation) { DogsOperations::Create.new(dogs_repo: repo, bark_operation: bark) }
    #...
  end
  ```
### Bad
- Don't call model methods that are database related. Use `Repository` instead.
  ```ruby
  module DogsOperations
    class Create
      #...
      
      def call(context:, params:)
        #...
        dog = Dog.create(dog_params) # Bad - Use DogsRepository#create instead
        #...
      end
      
      #...
    end
  end
  ```
- Don't add to many different actions in one operation. Divide it to smaller operations and call them. Keep your operations rather small and clean
  ```ruby
  module DogsOperations
    class Create
      #...
      
      def call(context:, params:)
        action_1() 
        action_2() 
        action_3() 
        #...
        action_42() # Bad - Divide to smaller operations
      end
      
      #...
    end
  end
  ```

## Controller
### Good
- Place here code responsible for calling operations and deciding what to render depending on results and happened errors.
  ```ruby
  class DogsController < ApplicationController
    #...
    def create
      @dog = DogOperations::Create.new.(params: params)
      render_dog()
    rescue UnauthorizedError => e
      handle_unauthorized_error(e)
    rescue ValidationError => e
      handle_validation_error(e)
    end

    #...
  end
  ```
- It's ok to have methods used in your views in `Controller` while you keep it clear
  ```ruby
  class DogsController < ApplicationController
    #...

    helper_method :can_create?

    private

    def can_create?
      #...
    end
  end
  ```
### Bad
- Don't call Repositories and Model methods from the controller. They should be called from Operations. Controller only get results of Operations and render it
  ```ruby
  class DogsController < ApplicationController
    def create
      @dog = Dog.create(params) # Bad - Call an Operation instead
      @dog = DogsRepository.create(params) # Bad - Call an Operation instead
    end
  ```
## View
### Good
- Use views as templates not as a place where you decide what to render
  ```haml
  .title
    = title
  .main
    - if can_create?
      = a
    - else
      = b
  ```
### Bad
 - Don't put any business logic in views
   ```haml
    .title
      = title
    .main
      - if current_user.can?(:whatever) # Bad - Create a method in controller and call it here
        = a
      - else
        = b
    ```
  - Don't use helpers for not global methods. Add methods to contoller and use `helper_method` instead
    ```ruby
    class DogsController < ApplicationController
      #...

      helper_method :can_create?

      private

      def can_create?
        #...
      end
    ```

