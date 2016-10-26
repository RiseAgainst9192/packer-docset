require 'rake'
require 'sequel'
require 'nokogiri'
require 'logger'
require 'pry'

task :default => [:build]

desc 'Build the docset'
task :build => [:clean, :download, :make_database, :build_docset, :compress_docset]

task :clean do
  system('rm -rf hts-* backblue.gif fade.gif index.html cookies.txt')
  system('rm -f docSet.dsidx')
  system('rm -rf packer.io')
end

task :download do
  puts 'Downloading documentation...'
  system('httrack http://www.packer.io/docs --mirror')
  system('httrack http://www.packer.io/docs --update')
  system('rm www.packer.io/index.html')
  system('cp www.packer.io/intro/index.html www.packer.io/index.html')
end

task :make_database do
  db = Sequel.connect('sqlite://docSet.dsidx', loggers: [Logger.new(STDOUT)])
  db.run('CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);')
  db.run('CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);')

  Dir.chdir('www.packer.io/docs/') do
    generate_entries(db, path: 'command-line/*.html',     type: 'Command',      title_sub: /\w+-\w+: /,  skip_file: 'introduction')
    generate_entries(db, path: 'provisioners/*.html',     type: 'Provisioner',  title_sub: ' Provisioner')

    generate_entries(db, path: 'templates/*.html',        type: 'Guide',        title_sub: ' Templates',      title_prefix: 'Template:')
    generate_entries(db, path: 'builders/*.html',         type: 'Guide',        title_sub: ' Builder',        title_prefix: 'Builder:')
    generate_entries(db, path: 'post-processors/*.html',  type: 'Guide',        title_sub: ' Post-Processor', title_prefix: 'Post-Processor:')
  end

  # Add guides
  db[:searchIndex] << { name: 'Installation',                   type: 'Guide', path: 'docs/installation.html' }
  db[:searchIndex] << { name: 'Terminology',                    type: 'Guide', path: 'docs/basics/terminology.html' }
  db[:searchIndex] << { name: 'Command-Line',                   type: 'Guide', path: 'docs/command-line/introduction.html' }
  db[:searchIndex] << { name: 'Templates',                      type: 'Guide', path: 'docs/templates/introduction.html' }
  db[:searchIndex] << { name: 'Core Configuration',             type: 'Guide', path: 'docs/other/core-configuration.html' }
  db[:searchIndex] << { name: 'Debugging',                      type: 'Guide', path: 'docs/other/debugging.html' }
  db[:searchIndex] << { name: 'Environment Variables',          type: 'Guide', path: 'docs/other/environmental-variables.html' }
  db[:searchIndex] << { name: 'Extend: Packer Plugins',         type: 'Guide', path: 'docs/extend/plugins.html' }
  db[:searchIndex] << { name: 'Extend: Developing Plugins',     type: 'Guide', path: 'docs/extend/developing-plugins.html' }
  db[:searchIndex] << { name: 'Extend: Custom Builder',         type: 'Guide', path: 'docs/extend/builder.html' }
  db[:searchIndex] << { name: 'Extend: Custom Command',         type: 'Guide', path: 'docs/extend/command.html' }
  db[:searchIndex] << { name: 'Extend: Custom Post-Processor',  type: 'Guide', path: 'docs/extend/post-processor.html' }
  db[:searchIndex] << { name: 'Extend: Custom Provisioner',     type: 'Guide', path: 'docs/extend/provisioner.html' }

  db.disconnect
end

task :build_docset do
  contents_dir  = 'Packer.docset/Contents'
  resources_dir = "#{contents_dir}/Resources"
  documents_dir = "#{resources_dir}/Documents"

  system("mkdir -p #{documents_dir}")
  system("cp -r www.packer.io/* #{documents_dir}/")
  system("cp Info.plist #{contents_dir}/")
  system("cp docSet.dsidx #{resources_dir}/")
  system('cp icon.png Packer.docset/')
end

task :compress_docset do
  system("tar --exclude='.DS_Store' -cvzf Packer.tgz Packer.docset")
end

def generate_entries(db, path:, type:, title_sub:, title_prefix: nil, skip_file: nil)
  Dir.glob(path).each do |filename|
    next if skip_file && filename.include?(skip_file)

    doc  = Nokogiri::HTML(File.open(filename))
    name = doc.css('h1').first.content.sub(title_sub, '')
    name = [title_prefix, name].join(' ') if title_prefix

    entry = { name: name, type: type, path: File.join('docs', filename) }
    puts entry
    db[:searchIndex] << entry
  end
end
