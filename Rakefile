#!/usr/bin/env rake
# encoding: utf-8

require 'yaml'
require 'fileutils'
require 'parallel'

CONFIG_FILE = 'config.yml'
BACKUP_DIR  = File.expand_path('./backups')
COMPUTER_NAME = `scutil --get ComputerName`.strip

# Prepare backup destination
Dir.mkdir BACKUP_DIR unless Dir.exist? BACKUP_DIR

# Load config
config = YAML.load_file CONFIG_FILE

desc "Clean a backup folder"
task :clean do
  print "Cleaning #{BACKUP_DIR} ... "
  begin
    FileUtils.rm_rf BACKUP_DIR
    Dir.mkdir BACKUP_DIR
  rescue
    puts "error"
  else
    puts "successful"
  end
end

desc "Backup task"
task :backup do |t, args|
  config.select!{ |n, a| ENV['only'].split(',').include?(n) } if ENV['only']
  config.reject!{ |n, a| ENV['without'].split(',').include?(n) } if ENV['without']

  error = []

  puts "# Processing"

  computer_path = File.join(BACKUP_DIR, COMPUTER_NAME)

  if FileTest.exist?(computer_path)
    FileUtils.rm_rf computer_path
    FileUtils.mkdir_p computer_path
  else
    FileUtils.mkdir_p computer_path
  end

  Parallel.map(config) do |name, args|
    store_dir = File.join(computer_path, name)
    FileUtils.mkdir_p store_dir

    backup_source = args["backup"]
    backup_script = args["backup_script"]

    proc_store_files = lambda do |src|
      expanded_path = File.expand_path(src)
      filename = File.basename(expanded_path).gsub(/^\./, 'dot_')

      if FileTest.exist?(expanded_path)
        begin
          FileUtils.cp_r expanded_path, File.join(store_dir, filename)
        rescue => e
          print 'x'
          error << [name, e]
        ensure
          print '-'
        end
      else
        print 'x'
      end
    end

    if backup_source
      if backup_source.kind_of? Array
        backup_source.each do |src|
          proc_store_files.call(src)
        end
      else
        proc_store_files.call(backup_source)
      end
    end

    proc_run_script = lambda do |script|
      if script =~ %r|{{destination}}|
        command_name = script.match(/^(.+?)\s\W/)[1].gsub(/\s/, '_')
        command = script.gsub %r[{{destination}}], %["#{File.join(store_dir, command_name)}"]
        begin
          system command
        rescue => e
          print 'x'
          error << [name, e]
        ensure
          print '-'
        end
      else
        print 'x'
        error << [name, "failed identicate: #{script}"]
      end
    end

    if backup_script
      if backup_script.kind_of? Array
        backup_script.each do |script|
          proc_run_script.call(script)
        end
      else
        proc_run_script.call(backup_script)
      end
    end

    sleep 0.1
  end

  puts "\n\nBackup complete"

  if error.size > 0
    puts "\n# Some error raised\n"

    error.each do |err_name, err_desc|
      puts "#{err_name}:"
      puts "    #{err_desc}"
      puts "--------------------"
    end
  end

end

desc "Restore task"
task :restore do
  computer_name = ENV['with'] || COMPUTER_NAME
  config.select!{ |n, a| ENV['only'].split(',').include?(n) } if ENV['only']
  config.reject!{ |n, a| ENV['without'].split(',').include?(n) } if ENV['without']

  error = []

  puts "# Processing"

  computer_path = File.join(BACKUP_DIR, computer_name)

  config.each do |name, args|
    store_dir = File.join(computer_path, name)
    store_path = File.join(store_dir, name)

    source = args["backup"]

    if source =~ %r|{{destination}}|
      print '+'
      error << [name, "command not to be restored: #{source}"]

    else
      begin
        puts "#{store_path} => #{File.expand_path(source)}"
        FileUtils.rm_rf File.expand_path(source)
        FileUtils.cp_r store_path, File.expand_path(source)
      rescue => e
        print '+'
        error << [name, e]
      ensure
        print '-'
      end
    end

    sleep 0.1
  end

  puts "\n\nRestore complete"

  if error.size > 0
    puts "\n# Some errors raised\n"

    error.each do |err_name, err_desc|
      puts "#{err_name}:"
      puts "    #{err_desc}"
      puts "--------------------"
    end
  end

end

desc "Compare"
task :compare do
  computer_name = ENV['with']
  target_name = ENV['to']

  unless computer_name
    puts "Usage: rake compare with=Knightsbridge to=zshrc"
    exit
  end

  if target_name
    source = File.join(BACKUP_DIR, computer_name, target_name, target_name)
    target = File.join(BACKUP_DIR, COMPUTER_NAME, target_name, target_name)
  else
    source = File.join(BACKUP_DIR, computer_name)
    target = File.join(BACKUP_DIR, COMPUTER_NAME)
  end

  puts "# Processing"
  puts target

  system "ksdiff", target, source
end
