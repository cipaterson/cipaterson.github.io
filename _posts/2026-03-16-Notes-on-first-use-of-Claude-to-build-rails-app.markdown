---
layout: post
title: "Notes on first use of Claude to build a rails app"
date: 2026-03-16
---

# Using Claude to create a rails app to manage sailings

Sunday, 08 March 2026

* * *

I generated a new rails app in the usual way:  
`bin/rails init sailings`

First time in Claude it suggested creating a CLAUDE.md with `/init`, so I did. Then I asked for best prompts to create some things:

1.  an entity
2.  an entity with controller and views  
    I just wanted to see what claude would do and for advice on how to ask.

Then I asked for a generate command to add rails 8 auth - it gave me the standard command and executed it for me (after confirmation)

I used rails console to add two users to the DB.

I added  
`<%= stylesheet_link_tag "https://cdn.simplecss.org/simple.css" %>`  
to application layout myself. I don't need Claude for the hard stuff...

At this point, localhost:3000 still shows the rails splash page, I need to set root in the routes.rb file to sailings#index. Which would then redirect to login page since not logged in. So first I need a sailing model and controllers, routes, and views to match.

```text
Create a Sailing resource. Generate: migration, model, controller, views, and routes.

  ## Fields
  - purpose: string
  - status: string
  - departs_at: datetime
  - returns_at: datetime
  - ln_contact: string
  - master: string
  - comments: text

  ## Model
  - Validations: purpose required, status in %w[draft active archived]
  - Associations: none

  ## Controller
  - Actions: index, show, new, create, edit, update, destroy
  - After create: redirect to show with notice
  - After destroy: redirect to index

  ## Views
  - index: table listing all records, link to new
  - show: display all fields, links to edit and back
  - new/edit: shared _form partial
  - _form: [describe each field's input type, e.g. "status as select"]
  - Use Turbo for form submissions (default in Rails 8)

  ## Routes
  resources :sailings

  Do not add pagination, search, API endpoints, JSON responses, or anything not listed above.
```

Claude created all the files needed (after confirmation) and the functionallity worked just as expected!

I added a root definition in routes.rb.

So, I can't logout nor create new users in the UI yet.

I asked claude create initial users for me, it created a seeds.rb file and ran rails db:seed.

Now for the many to many association:

```text
❯ Generate a Best practice prompt to create a many-to-many relationship between existing models Sailing and User where the join model has a status field: %w[EOI Accepted Not-required].                                                                                    
⏺ Here's the prompt:                                                                                                                                                                                                       Add a many-to-many relationship between Sailing and User through a SailingParticipant join model.           

  Join Model: SailingParticipant                                                                              

  Fields:                                                                                                                                        
  - sailing_id: references sailings (foreign key, not null)                                                                                      
  - user_id: references users (foreign key, not null)
  - status: string

  Validations:
  - sailing_id and user_id: presence required
  - status: inclusion in %w[EOI Accepted Not-required]
  - Uniqueness of user_id scoped to sailing_id (no duplicate participants per sailing)

  Migration:
  - Add foreign keys with null: false
  - Add a unique index on [:sailing_id, :user_id]

  Associations

  In SailingParticipant:
  - belongs_to :sailing
  - belongs_to :user

  In Sailing:
  - has_many :sailing_participants, dependent: :destroy
  - has_many :users, through: :sailing_participants

  In User:
  - has_many :sailing_participants, dependent: :destroy
  - has_many :sailings, through: :sailing_participants

  Generate

  - Migration
  - Model with validations and associations
  - Update Sailing and User models with associations

  Do not add a controller, views, routes, or anything not listed above.
```

I said "yes" and it generated everything just as expected.

```text
Generate a best practice prompt to manage sailing participants.  Add a link to each row of the sailing\#index view that goes to the manage
sailing participants view. There I want to be able to change the status of the sailing participant, or add a new sailing participant.

⏺ Here's the prompt:

  Add a SailingParticipantsController nested under Sailing to manage participants for a given sailing.

  Routes

  Nest under sailings, only the needed actions:
  resources :sailings do
    resources :sailing_participants, only: [:index, :create, :update], shallow: false
  end

  Controller: SailingParticipantsController

  - before_action: load @sailing from params[:sailing_id]
  - index: load all @sailing_participants for the sailing (include :user), and a new @sailing_participant for the add form
  - create: build participant on @sailing, redirect to index with notice on success, re-render index on failure
  - update: find participant on @sailing, update status, redirect to index with notice on success, re-render index on failure

  Views

  sailing_participants/index.html.erb:
  - Heading: "Participants for [sailing purpose]"
  - Table of current participants: user email, status, inline "Edit status" form (select + submit button per row using form_with patching the
  participant)
  - Below the table: an "Add Participant" form with a user select (all users not already participating) and a status select, posting to create

  No separate new/edit/show views — everything on the index page.

  Form details

  - Status select options: SailingParticipant::STATUSES
  - User select options: scoped to users not already a participant on this sailing
  - Use form_with (Turbo-compatible, no local: true)

  Sailing\#index view update

  Add a "Manage Participants" link to each row in app/views/sailings/index.html.erb pointing to sailing_sailing_participants_path(sailing).

  Do not add pagination, search, JSON responses, or anything not listed above.
```

⏺ Done. Here's what was created/updated:

| File | Change |
| --- | --- |
| config/routes.rb | Nested sailing_participants under sailings (index, create, update) |
| app/controllers/sailing_participants_controller.rb | New controller |
| app/views/sailing_participants/index.html.erb | Participant table with inline status update + add participant form |
| app/views/sailings/index.html.erb | "Manage Participants" link added to each row |

The status select menu auto-submits on change via onchange: "this.form.requestSubmit()", so no button click is needed to update a participant's  
status.

Adding my registrations:

```text
create a prompt to add a "My Registrations" controller.  It should show the sailings that the current logged in user is participating in.  It should allow the user to cancel the participation request.

⏺ Here's the prompt:

  ---
  Add a MyRegistrationsController to show the current user's sailing participations and allow cancellation.

  Routes

  resource :my_registrations, only: [:show, :destroy]

  Use resource (singular) since it represents the current user's own registrations.

  For destroy, use a member route on sailing_participants:
  resources :sailing_participants, only: [:destroy]
  (top-level, not nested)

  Controller: MyRegistrationsController

  - show: load all SailingParticipant records for Current.user, including :sailing
  - destroy: not needed here — cancellation is handled by SailingParticipantsController\#destroy

  SailingParticipantsController#destroy

  Add a destroy action:
  - Find the participant scoped to Current.user (not @sailing) so users can only cancel their own
  - Destroy it
  - Redirect to my_registrations_path with notice

  View: my_registrations/show.html.erb

  - Heading: "My Registrations"
  - Table: sailing purpose, departs_at, returns_at, status, and a "Cancel" button (DELETE to sailing_participant_path(participant), confirm
  prompt)
  - If no registrations: display "You have no registrations."

  Assumptions

  - Current.user is available (standard Rails 8 authentication generator pattern)
  - No new model changes needed

  Do not add an index action, admin controls, or anything not listed above.
```

Fleshing out the users model and adding a controller to manage them:

```text
create a prompt to add a Users Controller. Add CRUD routes. add a link in the navbar to the index view.

⏺ Here's the prompt:

  ---
  Add a UsersController with full CRUD for managing users.

  Routes

  resources :users

  Controller: UsersController

  - Actions: index, show, new, create, edit, update, destroy
  - After create: redirect to show with notice
  - After update: redirect to show with notice
  - After destroy: redirect to index with notice
  - Strong params: permit email_address, password, password_confirmation, first_name, last_name, mobile_phone, home_phone, birth_date,
  occupation, membership_type, sailing_class, sit_date

  Views

  - index: table listing all users with columns: email, first name, last name, membership type. Row links name to show.
  - show: display all fields, links to edit and back
  - new/edit: shared _form partial
  - _form:
    - email_address: email field
    - password and password_confirmation: password fields (only shown on new; on edit, leave blank to keep existing)
    - first_name, last_name, occupation, membership_type, sailing_class: text fields
    - mobile_phone, home_phone: telephone fields
    - birth_date, sit_date: date fields

  Navbar

  Add a "Users" link to app/views/layouts/application.html.erb pointing to users_path.

  Do not add authentication guards, pagination, search, or anything not listed above.
```

Oops for got to specify type of Membership type:

```text
Create a prompt to change the User field "Membership Type" to a string based enum just like Sailing.status. Membership Type should include
%w[Life Family Individual Junior]

⏺ Here's the prompt:

  ---
  Change User#membership_type to a string-based enum validated with inclusion, following the same pattern as Sailing#status.

  Model

  In app/models/user.rb, add:

  MEMBERSHIP_TYPES = %w[Life Family Individual Junior].freeze
  validates :membership_type, inclusion: { in: MEMBERSHIP_TYPES }, allow_blank: true

  View

  In app/views/users/_form.html.erb, replace the text_field for membership_type with:

  <%= f.select :membership_type, User::MEMBERSHIP_TYPES, { include_blank: "Select type" } %>

  No migration needed — the column is already a string.

  Do not change anything else.
```

Added a tabbed interface to the User#edit view:

```text
 Create a prompt to enhance the User#edit view with tabs to separate related attributes on different tabs. The tabs should be: "Personal",      
"Membership", "Next of Kin", "Training", "Qualifications".  Move email_address, First name, Last name, mobile phone, home phone, birth date,     
Occupation to the "Personal" tab.  Move Membership Type, Class, SIT Date to the "Membership" tab.                                                

⏺ Here's the prompt:

  ---
  Enhance app/views/users/_form.html.erb to use tabs to separate fields across multiple sections. Use a pure HTML/CSS tab implementation via a
  hidden radio input pattern — no JavaScript, no external libraries, no Stimulus controller.

  Tabs

  - Personal: email_address, first_name, last_name, mobile_phone, home_phone, birth_date, occupation
  - Membership: membership_type, sailing_class, sit_date
  - Next of Kin: empty for now, just show a placeholder "Coming soon."
  - Training: empty for now, just show a placeholder "Coming soon."
  - Qualifications: empty for now, just show a placeholder "Coming soon."

  Implementation

  Use the radio-button CSS tab pattern:

  <!-- one per tab -->
  <input type="radio" name="tab" id="tab-personal" hidden checked>
  <label for="tab-personal">Personal</label>
  ...
  <!-- tab content divs shown/hidden via CSS sibling selectors -->

  Add the required CSS directly in the _form.html.erb partial inside a <style> tag. Keep the CSS minimal: just enough to show/hide panels and
  style the active tab label.

  Notes

  - Password fields (shown only on new) remain above the tabs, outside the tab structure
  - The form submit button remains below the tabs, outside the tab structure
  - Do not create a new CSS file, Stimulus controller, or any JavaScript

```

Actually this tabbed both the edit and new views (they share a \_form partial). So the new view still needs to be refined.

Monday, 09 March 2026

* * *

Lot's of small changes made cleaning up CSS, etc.

Now for a big one:

```text
create a best practice prompt to change the SailingParticipant#index view to add a submit and cancel option and remove the onchange.  In the controller's update method I want to be able to detect when changes have been made to a participant's status and send them an email about that change. 
⏺ Here's the prompt:                           
  ---                                                                                                                                            
  Update SailingParticipantsController#update and the sailing_participants/index view to replace the auto-submit status select with an explicit save/cancel flow, and send a mailer when a participant's status changes. 
                 
  View: sailing_participants/index.html.erb

  Replace the inline onchange auto-submit form per row with an explicit form that has Save and Cancel buttons:

  - Remove onchange: "this.form.requestSubmit()" from the select
  - Add a "Save" submit button
  - Add a "Cancel" link that resets to the current status — link back to sailing_sailing_participants_path(@sailing)

  Controller: SailingParticipantsController#update

  Before updating, capture the old status. After a successful update, if the status changed, send a mailer notification:

  old_status = @sailing_participant.status
  if @sailing_participant.update(sailing_participant_params)
    if old_status != @sailing_participant.status
      SailingParticipantMailer.status_changed(@sailing_participant, old_status).deliver_later
    end
    redirect_to ...
  end

  Mailer: SailingParticipantMailer

  Generate with bin/rails generate mailer SailingParticipantMailer status_changed.

  - status_changed(sailing_participant, old_status): email the participant's user
    - To: sailing_participant.user.email_address
    - Subject: "Your registration status has been updated"
    - Body: include sailing purpose, old status, new status

  Mailer views

  - status_changed.html.erb: HTML email with the details
  - status_changed.text.erb: plain text version

  Do not change the create action, add action mailers, or modify anything else.
```

That is not what I want! I don't want to cancel or save changes to one participant's status - I want to change the statuses and then bulk save them back for a diff and sending emails.

```text
That's not quite what I want.  Please change it from a save per particpant row to allow changes to all the participants followed by a single   
form submit action.  In the controller, iterate over all participants and send an email for each one that has changed.                           
                          
⏺ Read 1 file (ctrl+o to expand)                                                                                            
⏺ I'll use a single form_tag posting to a bulk update action. Let me update the routes, controller, and view:

```

I pushed the repo to github before allowing these changes, just in case. But they seem to have done the trick. (Except that the AI forgot he removed the update method and added a bulk_update, this meant an error in the before_action)

Adding the "register button" to the sailings index view:

```text
create a best practice prompt to add a link on each row of the Sailing#index view.  This should be called "Register" if the current user is not registerred as a participant for the sailing, or "Unregister" if they are.  If register, then link to create a new sailing_participant with a default status of "EOI".  If unregister, then link to delete the sailing_participant.

⏺ Here's the prompt: 
  ---                                                                                                                                            
  Add a "Register" / "Unregister" link to each row in sailings#index based on whether the current user is already a participant in that sailing.

  Controller: SailingsController#index

  Preload the current user's participant records for all displayed sailings in a single query to avoid N+1:

  @my_participants = Current.user.sailing_participants
                           .where(sailing_id: @sailings.map(&:id))
                           .index_by(&:sailing_id)

  View: sailings/index.html.erb

  For each row, look up whether the current user has a participant record:

  my_participant = @my_participants[sailing.id]

  - If my_participant is nil: render a button_to "Register" that POSTs to sailing_sailing_participants_path(sailing) with params: {
  sailing_participant: { status: "EOI" } }
  - If my_participant exists: render a button_to "Unregister" that DELETEs to sailing_participant_path(my_participant) with a confirm prompt

  Notes

  - Use button_to for both actions (POST and DELETE require non-GET requests)
  - The existing SailingParticipantsController#create already handles the POST and sets status from params
  - The existing SailingParticipantsController#destroy already handles the DELETE
  - After register, the user is redirected to the participants index — change the redirect in create to go back to sailings_path with notice
  instead, so the flow stays on the index

  Do not add a new controller action, new route, or anything not listed above.
```

Register show's the sailing_participants#new view? I wanted it to go to create and register without any UI. The prompt seems to say it will do the right thing but maybe there's a bug somewhere.  
After a lot of head scratching I prompted with this:

```text
 There's an error.  The create method needs a user_id to build a sailing_participant and there is no user_id parameter in the request generated by the "Register" button.  Where should the current_user_id be added to correct this error?                                                     

⏺ The user_id should be set in the controller from Current.user, not trusted from params. Update SailingParticipantsController#create:           

⏺ Update(app/controllers/sailing_participants_controller.rb)                                                                                     
  ⎿  Added 1 line, removed 1 line                                                                                                                
       7    end                                                                                               
       8                                                                                                      
       9    def create                                                                                        
      10 -    @sailing_participant = @sailing.sailing_participants.build(sailing_participant_params)          
      10 +    @sailing_participant = @sailing.sailing_participants.build(sailing_participant_params.merge(user_id: Current.user.id))             
      11      if @sailing_participant.save                                                                    
      12        redirect_to sailings_path, notice: "You have been registered for this sailing."               
      13      else                                                                                                      
⏺ Done. user_id is now set server-side from Current.user rather than relying on a param, which is also more secure.  
```

I asked this:  
`I want to produce a PDF form of a sailing together with the users participating. What's the best way to structure this?`

It recommend Prawn, an all Ruby solution and gave it's reasons. Crazy.

Ran the prompt and bingo - I have a sailing plan PDF.

Wednesday, 11 March 2026

* * *

Time to add role based authorization:

```text
I want to implement role based authorisation.  Roles sholud be "Member", "Office Staff", "Crewing Operator", "Trainer", "Purser".  A User can hold      
multiple roles.  Show me a prompt to add this to the User model.  Advise how to enable/disable GUI elements and functionality depening on the role held   
by the current user.                                                                                          
⏺ Searched for 2 patterns, read 2 files (ctrl+o to expand)                                                    
⏺ Here's everything you need.                                                                                                 
  ---                                                                                                                                                     
  Implementation Prompt                                                                                                   
  Use this as your instruction to implement the feature:                                                           
  Implement role-based authorization on the User model. Roles are: member, office_staff, crewing_operator, trainer, purser. A user can hold multiple roles
   simultaneously.

  Use a bitmask integer column (roles_mask) on the users table — no extra tables needed, and no gems required.

  Steps:
  1. Generate a migration to add roles_mask :integer, default: 0, null: false to users.
  2. Add ROLES constant and bitmask helper methods to User:
    - roles — returns array of role name strings
    - roles= — accepts array of strings, sets mask
    - has_role?(role) — boolean check
    - Predicate methods: member?, office_staff?, crewing_operator?, trainer?, purser?
    - admin? convenience alias for office_staff? || crewing_operator? (or define as needed)
  3. Add a scope :with_role(role) on User for querying.
  4. Add a multi-checkbox form control to the user edit view so roles can be assigned.
  5. Permit roles: [] in the users controller strong params.
  6. Write model tests covering bitmask logic.

  ---
  The Bitmask Pattern (for reference)

  # In User model
  ROLES = %w[member office_staff crewing_operator trainer purser].freeze

  def roles
    ROLES.select { |r| roles_mask & 2**ROLES.index(r) != 0 }
  end

  def roles=(selected)
    self.roles_mask = (selected.map(&:to_s) & ROLES).sum { |r| 2**ROLES.index(r) }
  end

  def has_role?(role)
    roles.include?(role.to_s)
  end

  ROLES.each do |role|
    define_method(:"#{role}?") { has_role?(role) }
  end

  def self.with_role(role)
    mask = 2**ROLES.index(role.to_s)
    where("roles_mask & ? != 0", mask)
  end

  ---
  Enabling/Disabling GUI Elements

  1. ApplicationController helper

  # app/controllers/application_controller.rb
  helper_method :current_user

  def require_role(*roles)
    unless roles.any? { |r| current_user&.has_role?(r.to_s) }
      redirect_to root_path, alert: "Not authorised."
    end
  end

  Use in controllers:
  before_action -> { require_role("office_staff", "crewing_operator") }, only: [:new, :create, :edit, :update, :destroy]

  2. View helpers for conditional UI

  # app/helpers/application_helper.rb
  def can?(role)
    current_user&.has_role?(role.to_s)
  end

  Then in views:
  <%= link_to "Edit Sailing", edit_sailing_path(@sailing) if can?("office_staff") || can?("crewing_operator") %>

  <% if can?("purser") %>
    <%= render "financial_tab" %>
  <% end %>

  3. For complex rules — a Policy object (no gem needed)

  # app/policies/sailing_policy.rb
  class SailingPolicy
    def initialize(user, sailing)
      @user = user
      @sailing = sailing
    end

    def edit?    = @user.office_staff? || @user.crewing_operator?
    def destroy? = @user.office_staff?
    def manifest? = @user.crewing_operator? || @user.trainer? || @user.purser?
  end

  In controllers:
  def policy = SailingPolicy.new(current_user, @sailing)
  before_action -> { redirect_to root_path unless policy.edit? }, only: [:edit, :update]

  ---
  Summary of approaches

  ┌───────────────────────────┬─────────────────────────────────┐
  │          Concern          │            Mechanism            │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Route/action protection   │ before_action with require_role │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Show/hide buttons & links │ can? helper in views            │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Complex multi-role rules  │ Policy object per resource      │
  ├───────────────────────────┼─────────────────────────────────┤
  │ Querying users by role    │ User.with_role("trainer")       │
  └───────────────────────────┴─────────────────────────────────┘

  This approach requires no gems (no Pundit/CanCanCan/Rolify) and fits naturally with your existing Rails 8.1 stack. Add Pundit later if policies proliferate.
```

Add some attributes to the Quals tab:

```text
 Add Attributes to the User model: ESS, MED, WWVP. Each attribute should consist of a string for the Qualification, a Date Issued, and a Date Expired. Add these fields to the Qualifications tab on the User form partial.  
```

Add some dates to the Training tab:

```text
Add attributes to the User model: Knots, Marine Safety Refresher.  All these fields are dates.  Place them all on the Training tab of the User form partial.
```

The generate PDF for the sailing manifest disappeared. I asked Claude and it searched the git log and told me I never committed it. Probably I hadn't committed and pushed before messing with the names of the repos. Easy to re-implement though.

**BUG**: Add Participant on manage_participants does not work. Possibly the ctrl'r:create() merges the current user ID when it should be using the selected user.id.

There are a few things that still need doing but only two are important:

- [x] Changing the names in GUI for some entities
- [ ] adding the rank attribute to the manage crew screen, which should impress people.

Adding secrets to allow prod to send emails:

```text
% EDITOR="zed --wait" rails credentials:edit --environment=production
Adding config/credentials/production.key to store the encryption key: d706fdbba13111111a39e8c5a1

Save this in a password manager your team can access.
```

I'm using Brevo.com to send emails - 300 free per month.

19:47 pm - still struggling....

Looking at the bug in adding a crew member in manage crew - Claude provided some code but got it wrong! It thought that an int needed to be converted to a string to match but it did not.

21:22 pm bug fixed and flash div's added to application template, and removed from specific views (mostly passwd and session).

Friday, 13 March 2026

* * *

Diagnosing email delivery. I've made some notes on the sorry story of setting up email below. Maybe I'll write a blog post for posterity as the info out there, documentation and stackoverflow, is pretty poor.

## Email Setup steps:

### Configure your email domain to ensure delivery

This involves adding records to DNS to authenticate emails originating from your domain. Mail delivery services (mailgun, Brevo, Resend, etc) won't try to deliver an email that would likely fail the recipient's checks. Be patient - it takes time for DNS records to propagate.

Brevo was able to reach out to namecheap for me (after I authenticated) and place the records automagically.

### Change default 'from:' email

In app/mailers/application_mailer.rb you can set the default from address to some email from the domain above. You can also override this in each separate mailer, or even on each email sent if you really want.

### Configure Send mail config

In config environments (esp. production.rb), set the correct smtp_settings. This probably involves secrets.

### Configuring secrets

Very confusing.  
https://jonsully.net/blog/rails-credentials-can-be-confusing

I'm still not sure exactly how the global credentials and the environment specific credentials interact, but I switched to using just the global credentials and key (and setting that key as the RAILS_MASTER_KEY fly.io secret) and not having separate dev/prod credential files. It's not clear to me what to set RAILS_MASTER_KEY to if I use the environment specific credentials config files. Now prod seems to be able to decode the smtp user and password ok.

I deleted all creds and key files (config/credentials is empty now) and used:  
`EDITOR=vim rails credentials:edit`  
to re-create and edit the global credentials, then:  
`fly secrets set RAILS_MASTER_KEY=$(cat config/master.key)`

### Make sure Solid Queue is running

```text
  To confirm it's actually running in production:
  fly ssh console -a sailings
  bin/rails console
  SolidQueue::Process.all   # should show registered worker processes
  If that returns records, Solid Queue is running.
```

There are other diagnostic steps mentioned below.

### Diagnosing

If no email arrives then follow these steps.

Test a raw curl request to check that the authentication is understood and that the port is open outwards from the environment.

```text
curl -s --ssl-reqd \
   --url 'smtp://smtp-relay.brevo.com:587'  \
   --user 'a4b136001@smtp-brevo.com:[xsmtpsib-token]' \
    --mail-from 'no-reply@sailings.zzyplza.com' \
    --mail-rcpt 'cipaterson@proton.me' \
    --upload-file /dev/null
```

Next check from rails. It helps to set these in config/environments/xxx.rb:

```text
config.action_mailer.raise_delivery_errors = true
config.action_mailer.delivery_method = :smtp
```

delivery method is supposed to default to :smtp but some say it's best to be explicit.

**NOTE! Not everything is reloaded automatically even in the development environment.**  
You must restart the server in these cases:  
changes to the config/environment are NOT automatically loaded.  
code changes in config/initializers.  
Files in config.autoload_once_paths: These are autoloaded but not reloaded

Once I realised this I finally got an error message that I could google.

A good trick is to increase the log level in production temporarily by setting an environment var:  
`fly secrets set RAILS_LOG_LEVEL=debug`

You can check manually the rails side in the console:

```text
PasswordsMailer.reset(User.find_by(email_address: "cipaterson@proton.me")).deliver_now
```

This sends the email and it arrives fine without the SSL CRL error I get in dev.  
https://dev.to/madhuhari188/how-we-solved-unable-to-get-certificate-crl-in-rails-a-debugging-story-2pna

It's also good to check config in the rails console:

```text
Rails.application.credentials.smtp
Rails.configuration.action_mailer.smtp_settings
```

This is to make sure that the secrets and credentials are being read correctly in production.

16:36 pm  
currently credentials are being read both in dev and prod (after setting RAILS_MASTER_KEY) correctly.  
And the above console invoking works (with deliver_now) BUT the normal rails flow does not. Perhaps a queue problem.

It seems dev does not use solid queue at all - it uses "async" rather than a queue and worker process(es). Prod is configured to use is by default.

Diagnosing solid queue:

```text
fly ssh console -a sailings
  bin/rails console
  SolidQueue::Job.count
  SolidQueue::Job.last(5)
  SolidQueue::FailedExecution.all
```

The README here is very useful, the Rails Guide "basics of Solid Queue" is less so:  
https://github.com/rails/solid_queue  
mentions how to set up solid queue in dev to match prod so it's useful to see the details of the set-up.

It mentions running `bin/jobs` after. I ran it for fun and suddenly the queue emptied and the jobs all ran - email's all over the place.  
Claude says this is not necessary - only needed for running sold core in a different process, it insisted that all this is handled in puma natively which is find for a small one container app. It said I can confirm the processes are running with:

```text
  To confirm it's actually running in production:
  fly ssh console -a sailings
  bin/rails console
  SolidQueue::Process.all   # should show registered worker processes
  If that returns records, Solid Queue is running.
```

I said that SolidQueue::Process.all returns an empty array - why is that? Finally it said:

```text
The problem is clear. deploy.yml is Kamal configuration — it has no effect on your Fly.io deployment. The SOLID_QUEUE_IN_PUMA: true env var is never set on Fly, so Puma starts without the Solid Queue plugin.                                                                                   
Confirmed — no SOLID_QUEUE_IN_PUMA in fly.toml. Also notice the start command is bin/rails server which bypasses the bin/thrust wrapper, but more importantly it never gets SOLID_QUEUE_IN_PUMA.
```

So we set it in the fly.toml, problem solved!

Prod now sends email fine. Dev still has the openssl/ruby SSL CRL problem but I don't care - I'll switch it back to using:  
`config.action_mailer.delivery_method = :test`

**BUG**: Noticed that emails are sent to users who have their status on a voyage changed but not to users who are added for the first time.

Saturday, 14 March 2026  
Changes:

1.  email works now
2.  Improved styling when viewing a voyage or a member
3.  Added pagination to Voyages list
4.  Added date filter that defaults to today - like the current sailings app
5.  Added the full set of columns to Voyages and My Registratons
6.  In Manage Crew - made adding a user default to status "Accepted"
7.  Also sends an email now when a user is added by the crewing officer
8.  Added a reminder to status changed emails about arriving early

All the above (except email of course) was done in about 3-4 hours with Claude.

That's it for this tale, it's been an interesting introduction to "vibe coding".
