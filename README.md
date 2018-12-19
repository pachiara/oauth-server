# README

Esempio di server provider oauth2 realizzato con gem 'devise' e 'doorkeeper'.


Il server gira su http://localhost:3000
prima di tutto occorre configurare l'applicazione client sul server:
http://localhost:3000/oauth/applications/new
fornendo nome dell'applicazione = oauth-client
e redirect-uri = http://localhost:3001/auth/doorkeeper/callback
confermando vengono fornite le chiavi Application UID e Secret
che devono essere impostate sul client in config/initializers/omniauth.rb

Il Flusso quando si visita la home del client (http://localhost:3001):
1) si viene rediretti su http://localhost:3001/auth/doorkeeper definito in routes.rb
   questo url fa partire l'autenticazione oauth utilizzando “doorkeeper” omniauth strategy
2) si viene sempre reindirizzati al server provider SSO definito nel client in lib/doorkeeper.rb
3) se non si e' autenticati al server si viene reindirizzati su server a localhost:3000/users/sign_in
4) dopo aver effettuato l'accesso si viene indirizzati sul client a /auth/doorkeeper/callback che
   corrisponde in routes a controller application metodo authentication_callback
5) in questo esempio vengono stampate in json le informazioni ricevute dal server

# Implementazione Server SSO

* rails new -d mysql oauth-server

cd oauth-server

* GemFile
```
gem "devise"
gem "doorkeeper"
```
* bundle install

* rails g devise:install
* rails g devise User

* rails g doorkeeper:install
* rails generate doorkeeper:migration

* rake db:create
* rake db:migrate

* config/initializers/doorkeeper.rb
```
resource_owner_authenticator do
  current_user || begin
    session[:user_return_to] = request.fullpath
    redirect_to new_user_session_url
  end
end
admin_authenticator do
  current_user || redirect_to(new_user_session_url)
  # solo gli utenti amministratori
  # current_user && current_user.admin? || redirect_to(new_user_session_url)
end
```
* db/seed.rb
```
  User.create! email: 'test@test.com', password: 'password', password_confirmation: 'password'
```
* rake db:seed

* application controller
```
class ApplicationController < ActionController::Base
  before_action :doorkeeper_authorize!, only: :me

  def me
    render json: User.find(doorkeeper_token.resource_owner_id).as_json
  end
end
```
* config/routes.rb
```
Rails.application.routes.draw do
  use_doorkeeper
  devise_for :users
  root :to => 'application#me'
  get '/me' => 'application#me'
end
```

* rails s

test
curl -I localhost:3000/me
