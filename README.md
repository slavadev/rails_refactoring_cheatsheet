# Rails Refactoring Cheatsheet
Based on [this article](http://link_will_be_here)

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
- Put in the model only validations, associations, and methods that actually are related to the model.
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
- Keep in the model all "extentions" you use
  ```
  class Dog < ApplicationRecord
    acts_as_paranoid
    #...
    dragonfly_accessor :image
  end
  ```

### Bad
- Use callbacks like `after_*` and `before_*`. Put this code in `Repository` if it is DB-related code and in `Operation` if it is business logic
  ```
  class Dog < ApplicationRecord
    before_save :calculate_size # Move it to Repository
    after_create :create_booth # Move it to Operation
  end
  ```
- Keep scopes in the model. Move them to `Repository`
  ```
  class Dog < ApplicationRecord
    scope :all, -> { where('age > 1') } # Move it to Repository

    # Move it to Repository
    def self.puppies
      Dog.where('age < 1')
    end
  end
  ```
- Use nested attributes for associations. Move creating, updating and deleting associations to `Operation`
  ```
  class Dog < ApplicationRecord
    accepts_nested_attributes_for :booth # Move this logic to Operation
  end
  ```
