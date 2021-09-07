## Тестовое задание на позицию DevOps Junior

### Задача:

	У нас мультизональный кластер (три зоны), в котором пять нод
	приложение требует около 5-10 секунд для инициализации.

	По результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой.

	На первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory.

	Приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём.

	- Хотим максимально отказоустойчивый deployment.
	- Хотим минимального потребления ресурсов от этого deployment’а.

### Решение:

*Для задачи использовался кастомный образ с Django и одним эндпоинтом для эмуляции нагрузки*

```views.py
class HomeView(TemplateView):
	"""
	Main endpoint for emulating application load.
	"""
	template_name: str = 'app/home.html'

	def get(
		self,
		request: HttpRequest,
	) -> HttpResponse:
		"""
		Multiplying numbers in the loop. 
		Loading CPU.
		Args:
			request: HttpRequest. Incoming
				user request.
		Return:
			HttpResponse: Django built-in HttpResponse 
				whose content is filled with the 
				result of calling 
				django.template.loader.render_to_string() 
				with the passed arguments. In this
				case no arguments presents.
		"""
		result: Generator[int, None, None] = [sqrt(i * x) for (i, x) in enumerate(range(0, 30000000))]
		
		return render(
			request,
			template_name=self.template_name,
			status=200,
		)
```

	У нас есть плавающая нагрузка. В пике справляются с ней 4 пода. Соответственно, когда пика нет, то 4 пода поддерживать не имеет смысла. Значит нам нужно масштабироваться от 1 до 4 подов.

	CPU у нас имеет два состояния, а память - одно. Значит нам надо привязываться к значениям по CPU. Тут нам понадобится выставить ресурсы для пода. CPU и memory укажем в requests (желаемые, минимальные значения), а в максимальных, limits, проставим только для memory. Так как значение у нас в "районе", то возьмем чуть больше 128М. Но в тестовом примере limit не был использован, чтобы не вызвать OOMKilled, так как при вычислении, создается дополнительное количество объектов в памяти.

	Необходимо настроить Horizontal Pod Autoscaler. Предположим, что у нас ноды с 1 CPU. Соответственно, 100 / 4 = 25. Выставляем на 20%, после чего HPA должен поднимать новые поды, при достижении этого значения CPU Utilization.

	Без нагрузки приложение потребляет 0.1 CPU. После пика три пода будут "убиты". По-умолчанию, через 5 минут, после падения нагрузки.

	Приложение инициализируется 5 - 10 секунд. Значит нужны helthcheck`и для проверки готовности приложения. Для этого будут использоваться LivenessProbe и ReadinessProbe.

	ReadinessProbe: первая проверка будет запущена через 5 секунд, после старта контейнера и будет выполнятся каждые 5 секунд. Используем httpGet. Если проба пройдет, контейнер будет принимать трафик.

	LivenessProbe: первая проверка выполнится через 15 секунд, таким образом, у ReadinesProbe будет одна - две проверки. В случае неуспеха контейнер перезапустится.

	Приложение должно быть доступно из браузера. Для этой цели использован NodePort.

	Стратегия в deployment используется по-умолчанию - RollingUpdate.
	Приложение изолировано mindbox-ns неймспейсом.

### Как запустить:

```terminal
$ kubectl create -f <path_to_mindbox_tz_folder>
```

### Источники:

1. https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
2. https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
3. https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
4. https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details
5. https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/


***Буду рад любой обратной связи***

[TG](https://t.me/coveraver)
&
[Mail](mailto:sivolapenko.valeriy@gmail.com)
