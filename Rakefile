require 'rspec/core'
require 'rspec/core/rake_task'
require 'json'
require 'yaml'
require 'fileutils'
require 'sequel'

# system database configuration
require_relative 'lib/system/database'
include Intrigue::System::Database

# Config files
$intrigue_basedir = File.dirname(__FILE__)
$intrigue_environment = ENV["INTRIGUE_ENV"] || "development"

# Configuration and scripts
procfile_file = "#{$intrigue_basedir}/Procfile"
config_directory = "#{$intrigue_basedir}/config"
puma_config_file = "#{$intrigue_basedir}/config/puma.rb"
system_config_file = "#{$intrigue_basedir}/config/config.json"
database_config_file = "#{$intrigue_basedir}/config/database.yml"
sidekiq_config_file = "#{$intrigue_basedir}/config/sidekiq.yml"
redis_config_file = "#{$intrigue_basedir}/config/redis.yml"
control_script = "#{$intrigue_basedir}/util/control.sh"

all_config_files = [
  procfile_file,
  puma_config_file,
  system_config_file,
  database_config_file,
  sidekiq_config_file,
  redis_config_file
]

desc "Clean"
task :clean do
  puts "[+] Cleaning up!"
  all_config_files.each do |c|
    FileUtils.mv c, "#{c}.backup"
  end
end

desc "Load Global Namespace"
task :load_global_namespace do
  puts "Pulling global namespace"
  require './core'
  Intrigue::Model::GlobalEntity.load_global_namespace(ENV["INTRIGUEIO_DATA_API_KEY"])
  puts "Done pulling global namespace"
end

desc "System Setup"
task :setup do

  puts "[+] Setup initiated!"

  ## Copy puma config into place
  puts "[+] Copying Procfile...."
  if File.exist? procfile_file
    puts "[ ] File already exists, skipping: #{procfile_file}"
  else
    puts "[+] Creating.... #{procfile_file}"
    FileUtils.cp "#{procfile_file}.default", procfile_file
  end

  ## Copy puma config into place
  puts "[+] Copying puma config...."
  if File.exist? puma_config_file
    puts "[ ] File already exists, skipping: #{puma_config_file}"
  else
    puts "[+] Creating.... #{puma_config_file}"
    FileUtils.cp "#{puma_config_file}.default", puma_config_file
  end

  ## Copy system config into place
  puts "[+] Copying system config...."

  if File.exist? system_config_file

    puts "[ ] File already exists, skipping: #{system_config_file}"
    config = JSON.parse(File.read(system_config_file))
    system_password = config["credentials"]["password"]

  else
    puts "[+] Creating system config: #{system_config_file}"
    FileUtils.cp "#{system_config_file}.default", system_config_file

    # Set up our password
    config = JSON.parse(File.read(system_config_file))
    if config["credentials"]["password"] == "AUTOGENERATED"
      # generate a new password
      system_password = "#{(0...11).map { ('a'..'z').to_a[rand(26)] }.join}"
      puts "[+] ==================================="
      puts "[+] GENERATING NEW SYSTEM PASSWORD!"
      # Save it
      config["credentials"]["password"] = system_password
      File.open(system_config_file,"w").puts JSON.pretty_generate(config)
    else
      system_password = config["credentials"]["password"]
    end
  end

  # Create SSL Cert
  if !(File.exist?("#{$intrigue_basedir}/config/server.key") && File.exist?("#{$intrigue_basedir}/config/server.crt"))
    puts "[+] Generating a new self-signed SSL Certificate..."
    Dir.chdir("#{$intrigue_basedir}/config/"){ 
      subject_name = "/C=GB/ST=London/L=London/O=Global Security/OU=IT Department/CN=intrigue.local"
      command = "openssl req -subj '#{subject_name}' -new -newkey rsa:2048 -sha256 -days 365 -nodes -x509 -keyout server.key -out server.crt"
      puts `#{command}`
    }
  else
    puts "[+] SSL Certificate already exists!"
  end

  ## Copy database config into place
  puts "[+] Copying database config."
  if File.exist? database_config_file
    puts "[ ] File already exists, skipping: #{database_config_file}"
  else
    puts "[+] Creating.... #{database_config_file}"
    FileUtils.cp "#{database_config_file}.default", database_config_file

    # Set up our password
    config = YAML.load_file(database_config_file)
    config["development"]["password"] = system_password
    config["production"]["password"] = system_password
    File.open(database_config_file,"w").puts YAML.dump config
  end

  ## Copy redis config into place
  puts "[+] Copying redis config."
  if File.exist? redis_config_file
    puts "[ ] File already exists, skipping: #{redis_config_file}"
  else
    puts "[+] Creating.... #{redis_config_file}"
    FileUtils.cp "#{redis_config_file}.default", redis_config_file
  end

  ## Place sidekiq task worker configs into place
  puts "[+] Setting up sidekiq config...."
  worker_configs = [
    sidekiq_config_file
  ]

  worker_configs.each do |wc|
    unless File.exist?(wc)
      puts "[+] Copying: #{wc}.default"
      FileUtils.cp "#{wc}.default", wc
    else
      puts "[+] #{wc} already exists!"
    end
  end
  # end worker config placement

  puts "[+] Downloading latest data files..."
  Dir.chdir("#{$intrigue_basedir}/data/"){ puts %x["./get_latest.sh"] }

  # Print it
  puts 
  puts "[+] ==================================="
  puts "[+] SYSTEM USERNAME: intrigue"
  puts "[+] SYSTEM PASSWORD: #{system_password}"
  puts "[+] ==================================="
  puts 

  puts "[+] Complete!"

end

namespace :db do
  desc "Prints current schema version"
  task :version do
    setup_database
    version = if $db.tables.include?(:schema_info)
      $db[:schema_info].first[:version]
    end || 0
    puts "[+] Schema Version: #{version}"
  end

  desc "Perform migration up to latest migration available"
  task :migrate do
    setup_database
    ::Sequel::Migrator.run($db, "db")
    Rake::Task['db:version'].execute
  end

  desc "Perform rollback to specified target or full rollback as default"
  task :rollback, :target do |t, args|
    setup_database
    args.with_defaults(:target => 0)
    ::Sequel::Migrator.run($db, "db", :target => args[:target].to_i)
    Rake::Task['db:version'].execute
  end

  desc "Perform migration reset (full rollback and migration)"
  task :reset do
    setup_database
    ::Sequel::Migrator.run($db, "db", :target => 0)
    ::Sequel::Migrator.run($db, "db")
    Rake::Task['db:version'].execute
  end
end
