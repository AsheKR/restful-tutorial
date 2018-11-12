# Installation

우분투 2.7을 사용 (pyenv shell system)
sudo apt install python-pip
pip install --user --upgrade pipenv
zsh 안의 `export PATH=$HOME/.local/bin:$PATH` 추가
pip install --upgrade pip
~/.local/bin/pipenv

Ubuntu
pip install을 해당 사용자 폴더에 실행
가상환경에 pip install 을 하는것이 아니라 자신의 홈폴더에 추가

# Execute

`pipenv --python <version>`을 통해 가상환경 설치

`pipenv shell`명령어를 통해 가상환경을 적용시킨다.
가상환경 적용 후 `pipenv install '패키지'`를 실행하면 Pipfile, Pipfile.lock이 생성된다.

`pipenv install --dev <Package>`를 사용하면 Dev-Packages에 설치된다.
