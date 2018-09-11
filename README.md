# Rails Refactoring Cheatsheet
Based on [this article](http://link_will_be_here)

## Table of Content
- [General](#general)
- [Model](#model)
- [Repository](#repository)
- [Operation](#operation)
- [Controller](#controller)
- [View](#view)

## General
- Add comments and documentation using [YARD](https://yardoc.org/). Check `.yardopts` file for custom tags that you can use. Add new if you need
  ```
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
- Try to write code that will be easy to test(create small methods that you can easy mock)
  ```
  # In main code
  def repo
    DogsRepository.new
  end

  def call(params)
    dog = repo.create(params)
    #...
  end

  # In specs
  let(:repo) { double('DogsRepository') }
  before do
    allow(create_operation).to receive(:repo).and_return(repo)
    allow(repo).to receive(:create).and_return(dog)
    # or
    allow(repo).to receive(:create).and_raise(ValidationError)
  end
  ```
- Follow some styleguide. Check your code with [Rubocop](https://github.com/rubocop-hq/rubocop)
- Ask your teammembers if you hesitating at any point

## Model
### Good
- Place in the model only validations, associations, and methods that actually are related to the model.
  ```
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
  ```
  class Dog < ApplicationRecord
    acts_as_paranoid
    #...
    dragonfly_accessor :image
  end
  ```

### Bad
- Don't use callbacks like `after_*` and `before_*`. Put this code in `Repository` if it is DB-related code and in `Operation` if it is business logic
  ```
  class Dog < ApplicationRecord
    before_save :calculate_size # Bad - Move it to Repository
    after_create :create_booth # Bad - Move it to Operation
  end
  ```
- Don't use scopes in the model. Move them to `Repository`
  ```
  class Dog < ApplicationRecord
    scope :all, -> { where('age > 1') } # Bad - Move it to Repository

    # Bad - Move it to Repository
    def self.puppies
      Dog.where('age < 1')
    end
  end
  ```
- Don't use nested attributes for associations. Move creating, updating and deleting associations to `Operation`
  ```
  class Dog < ApplicationRecord
    accepts_nested_attributes_for :booth # Bad - Move this logic to Operation
  end
  ```
- Don't call Repositories and Operations from the model. Repositories should be called from Operations and Operations should be called from other operations or layer above them(Controller, API, Console etc.)
  ```
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
  ```
  class DogsRepository
    #...
    def all()
      Dog.where('age > 1.0') # was a default scope
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
      dog = Dog.new(params)
      dog.size = dog.get_size_from_breed_and_age # was a callback before
      raise ValidationError, dog unless dog.save
      dog
    end
  end
  ```
- It's ok to have different repositories for one model
  ```
  class Dogs::SmallDogsRepository
    #...
  end
  
  class Dogs::OldDogsRepository
    #...
  end
  ```
## Bad
- Don't change other models in the repository of the model. It should be done in `Operation` and use another `Repository`
  ```
  class DogsRepository
    #...
    def create(params)
      dog = Dog.create(params)
      booth = Booth.create(dog) # Bad - Move it to operation and use another repository
      #...
    end
  end
  ```
- Don't call other Repositories and Operations from the repository. Repositories should be called from Operations and Operations should be called from other operations or layer above them(Controller, API, Console etc.)
  ```
  class DogsRepository
    def create(params)
      dog = Dog.create(params)
      booth = BoothsRepository.new.create(dog: dog) # Bad - Move to Operation
      DogsOperations::Bark.new.call(dog) # Bad - Move to Operation
      #...
    end
  end
  ```
